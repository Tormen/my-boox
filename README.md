# my-boox

Send, list, and delete files on an Onyx BOOX e-ink device via BOOXDrop —
from the command line.

A single self-contained Python script. No manual dependency installation:
on first run it creates its own virtualenv (under `~/.python.venv/my-boox`)
and installs everything it needs into it, then re-execs itself under that
interpreter.

## v3.0 — breaking change, no backwards compatibility

Earlier versions read and wrote Onyx's legacy `api/1/push/message` listing.
That listing is confirmed to drift out of sync with reality — it keeps
showing files long after they're genuinely deleted, and at least one file
had a completely different id there than its real delivery-layer document.

**v3.0 drops `push/message` entirely** and reads/writes `/neocloud/`
directly (the actual Couchbase Sync Gateway instance that drives real
device delivery), backed by a small local cache so `ls` stays fast. IDs
from a v2.x `ls` do not work with this version.

**If you used an earlier version, run the one-time cleanup first**
(see [Migrating from v2.x](#migrating-from-v2x) below) before switching.

## Origin

This started as a shell wrapper around
[hrw/onyx-send2boox](https://github.com/hrw/onyx-send2boox) (MIT licensed,
© 2022 Marcin Juszkiewicz), then absorbed and rewrote that project's logic
directly (its `boox.py`, `send_file.py`, `obtain_token.py`,
`request_verification_code.py`, and `delete_files.py`) into one file.

Notable fixes and discoveries along the way:

- **Proper auth-failure detection.** The upstream code assumed a response
  shape and blew up with a raw `TypeError: 'NoneType' object is not
  subscriptable` on any auth failure — and, due to a logic bug, an
  explicitly-passed fresh verification code was silently ignored in favor
  of whatever stale token already sat in the config file, meaning a "fresh
  login" could never actually happen. `my-boox` checks the HTTP status and
  the API's own `result_code` field, raises a specific exception on real
  auth failure, and always honors an explicit code over a stored token.
- **Automatic re-authentication.** On an auth failure, it requests a fresh
  verification code, prompts you for it, obtains a new token, saves it, and
  retries the action once.
- **The exact call sequence the BOOXDrop web client uses**, taken from HAR
  captures of a working browser upload and delete:

  | command | sequence |
  | --- | --- |
  | `send` | OSS upload (https) → `_bulk_docs` (`new_edits=false`) → `push/saveAndPush` **with `cbMsg`** naming that document |
  | `del` | `_bulk_docs` tombstone → `push/message/batchDelete` with the same id |

  Both halves of each are load-bearing:

  - **`cbMsg` on `saveAndPush`** is what makes the server *adopt* the
    document we just wrote instead of registering a second one of its own.
    Omitting it produced two `/neocloud/` documents per file (two rows in
    the web UI, one row here only because `ls` grouped them) — and a
    delete that removed one and left the other. With it, one file is one
    document, exactly as in the browser.
  - **`batchDelete`** removes the paired push-message record. The
    `/neocloud/` tombstone alone leaves it behind.

  Files sent by a pre-v4.0 `my-boox` still carry that stray second
  document. `ls` groups by (name, size) so they show as one row, and `del`
  expands each id to its whole group, so both go.
- **Bulk endpoints everywhere they exist (v4.1).** Wall-clock time is
  dominated by `api_pacing_seconds` (1.5s between requests, which is what
  keeps Onyx's backend from throwing `504`s), so the win is making fewer
  requests rather than pacing them harder:

  | command | before | now |
  | --- | --- | --- |
  | `ls`, N new/changed docs | `3 + N` | `1` warm, `3` cold |
  | `del`, N ids | `1 + 3N` | `3` |
  | `send`, N files | `~5N` | `N + 4` |

  - `_changes?include_docs=true` returns every document body inline, so
    the per-document `GET` is gone. (`_bulk_get`, which the browser uses,
    answers as `multipart/mixed` — HTTP 406 in the capture — and would
    need a MIME parser for the same data.)
  - `_bulk_docs` takes the whole batch in one request, for both the
    document writes on `send` and the tombstones on `del`; `batchDelete`
    already took an `ids` list. Neither sends the `_revs_diff` the browser
    posts first: it is a read-only query ("which of these revisions do you
    not already have?") whose answer we discard, so it cannot affect the
    write — and it was measured costing 18s of a 28s send when it hit a
    cold-cache 504. It is left commented in `write_docs_batch`, because it
    *is* what the browser does: if a delivery problem ever traces back to
    this, put it back first.
  - `uid` and the sync-gateway session are cached in the local cache file
    (never the token itself — only an 8-character tail, to key the entry),
    so a warm `ls` is a **single** request. A rejected session is refetched
    once automatically rather than failing the command.
  - `users/getDevice` and `im/getSig` run once per run, not once per file.
  - `_changes` is bounded with `limit=1000` (as the browser bounds it) and
    paged until a short page arrives. Steady state is still one request:
    nothing changed, so the first page is empty and short.

  The gateway intermittently hangs ~15s on a `_changes` and returns `504`,
  after which a retry answers in ~0.13s — the retry has never once been
  slow. That looks like a request landing on a cold backend, so the first
  retry now goes out **immediately** rather than sleeping the configured
  backoff first. Bounding the feed with `limit` did *not* demonstrably stop
  the hang; it is kept because it matches the browser and caps how much
  work a single request can ask for.

  Measured against a live account: 3-file `send` 15.7s, cold `ls` 3.7s,
  warm `ls` 0.57s, 3-file `del` 3 requests. Sent files were confirmed to
  appear on the device and to vanish again after `del`.
- **A central pacing/retry layer.** Every outgoing request — across `send`,
  `ls`, and `del` alike — goes through one function that paces requests and
  retries transient failures (`502`/`503`/`504`, connection errors,
  timeouts — these happen even in the real web client) with backoff,
  configurable via `api_pacing_seconds`/`api_retry_attempts`/
  `api_retry_delay_seconds`.
- **`/neocloud/` as the sole source of truth for `ls`/`del` (v3.0).**
  Cross-checking against Onyx's legacy `push/message` listing repeatedly
  surfaced divergence — entries that were actually already deleted, and at
  least one file whose legacy id didn't match its real `/neocloud/`
  document id at all. Rather than keep patching around that with fallback
  reconciliation logic, v3.0 reads and writes `/neocloud/` directly via a
  local incremental cache for listing and deletion, so there's nothing
  left to diverge from. `send` still also calls `saveAndPush`, per above —
  that one, unlike the listing, is genuinely load-bearing.

## Install

Just download the script and make it executable — there's nothing else to
install ahead of time:

```sh
curl -O https://raw.githubusercontent.com/<you>/my-boox/main/my-boox
chmod +x my-boox
mv my-boox /usr/local/bin/   # or wherever you keep your my-* scripts
```

The first run sets up `~/.python.venv/my-boox` automatically (installs
`requests` and `oss2` into it) and re-execs itself under it. Every run
after that is instant.

## Migrating from v2.x

Run the one-time cleanup script once, before using v3.0 for the first
time. It deletes every live `/neocloud/` document and everything
`push/message` still lists (both are wiped, since v3.0 doesn't track a
mapping between the two), so you start from a genuinely empty, clean
account rather than mixing old and new bookkeeping:

```sh
~/.python.venv/my-boox/bin/python3 cleanup_v3_migration.py "$(my-boox --config)"
```

**This permanently deletes data from your Onyx account — review the
script first.** It also removes the old `<config>.neocloud-state.json`
file, which v3.0 doesn't use.

## Configure

```sh
my-boox --create-config              # print a template to stdout
my-boox --create-config ~/.my-boox.conf # write it to a specific path
```

Template:

```ini
[default]
email =
token =
cloud = eur.boox.com
api_pacing_seconds = 1.5
api_retry_attempts = 3
api_retry_delay_seconds = 3.0
api_timeout_seconds = 30.0
duplicate_check_on_send = true
cache_path =
```

Fill in `email` with the address tied to your Onyx account. Leave `token`
blank — it gets filled in automatically the first time you run a command
and go through the re-auth prompt. `cloud` is your BOOXDrop server region
(`eur.boox.com` for EU, `push.boox.com` for US/VN).

- `api_pacing_seconds` — minimum interval between **any** two requests this
  tool makes (`send`, `ls`, `del` alike, and every request within each of
  them). Onyx's backend throws more `502`/`503`/`504` errors under
  back-to-back requests; this reduces how often that happens. `0` disables
  pacing.
- `api_retry_attempts` — how many times to retry any single request after a
  transient server error (`502`/`503`/`504`, connection errors, timeouts)
  before giving up on it.
- `api_retry_delay_seconds` — base backoff delay between retries
  (multiplied by the attempt number, so `3.0` gives 3s, then 6s).
- `api_timeout_seconds` — how long to wait for one request before giving up.
  Default `30.0`, and **lowering it is a trap**. When `/neocloud/_changes`
  misses its server-side cache — reliably the **first read after a write** —
  the gateway hangs ~15s, returns `504`, and a retry then answers in ~0.13s:
  receiving that `504` is what marks the work finished. Hanging up at 8s was
  measured and is strictly worse — the upstream keeps grinding, every retry
  hits the same cold cache, and the call fails after burning the whole retry
  budget.

  Only **safe** calls are retried. `push/saveAndPush` and the two
  verification-code calls are never replayed automatically: a `504` means the
  gateway stopped waiting, not that the server did nothing, so a blind retry
  could register the same push twice. Those failures say so and point you at
  `my-boox ls`.
- `duplicate_check_on_send` — `true`/`false`. When `true` (default),
  `send` skips any file whose name and size already match something on
  your account. Set `false` to always upload regardless.
- `send_skip_optional_calls` — `true`/`false`, default `false`. `send`
  makes two calls (`users/getDevice`, `im/getSig`) whose responses are
  discarded; they exist only because the upstream project mirrored the web
  client's sequence. Setting this `true` skips them, saving two requests
  (and two pacing intervals) per file. **Unverified** — a similarly
  redundant-looking call (`saveAndPush`) turned out to be required for
  delivery, so test on-device before trusting it for a real batch.
- `cache_path` — where the local `/neocloud/` mirror is kept. Blank
  (default) uses the config path with its extension replaced by
  `.cache.json` (so `~/.my-boox.conf` -> `~/.my-boox.cache.json`). Set an
  absolute path to use somewhere else — e.g. to share one cache across
  multiple config files, or keep it off a synced/backed-up directory. See
  [Local cache](#local-cache) below for the file's structure.

### Config file search order

`my-boox` looks for its config in this order:

0. `$MY_BOOX_CONFIG`, if set and non-empty — wins outright
1. `/LINKS/default/my-boox.conf`
2. `~/.my-boox.conf` — **dot-prefixed**, home only
3. `/etc/my-boox.conf`
4. `/usr/local/etc/my-boox.conf`

First match wins; running a command with no config anywhere exits `206`.
A `--config PATH` naming a file that doesn't exist also exits `206` rather
than silently falling back to the search.

The config stores your account `token`, which is a full credential — anything
that can read the file can send to and delete from your device. `my-boox`
writes it `0600`. Tighten an older one with `chmod 600 ~/.my-boox.conf`.

Check where it's currently resolving to:

```sh
my-boox --config
```

Or point at a specific file for one run:

```sh
my-boox --config /path/to/other.ini ls
```

## Local cache

By default, `<config path>.cache.json` mirrors `/neocloud/`'s
contents. `ls` refreshes it incrementally on every call — one cheap
`_changes` request covering everything since the last known cursor, then a
body fetch only for genuinely new or changed documents. The first `ls`
after a clean slate pays the cost of fetching every file's body once;
every call after that is cheap. Set `cache_path` in the config to use a
different location. Delete the cache file (or empty its `"docs"` object,
keeping `"last_seq": "0"`) to force a full resync from scratch.

`del` and `send` both update the cache directly rather than waiting for
the next `ls`, so it stays accurate across a session.

The file itself is JSON shaped like:

```json
{
  "last_seq": "<opaque _changes cursor, don't edit>",
  "docs": {
    "<32-char hex neocloud document id>": {
      "name": "<original filename>",
      "size": 123456,
      "rev": "<current CouchDB-style revision, e.g. '1-abcdef...'>",
      "createdAt": 1785761145930,
      "updatedAt": 1785761145930
    }
  }
}
```

`createdAt`/`updatedAt` are millisecond epoch timestamps and may be `null`
for entries the cache hasn't fully synced yet.

## Use

```sh
my-boox send some-document.pdf
my-boox send a.pdf b.pdf c.pdf        # multiple files in one call
my-boox ls
my-boox del <id>
my-boox del <id1> <id2>
```

With multiple files, `send` keeps going even if one fails (missing file,
transient server error, etc.) — failures are summarized at the end and the
exit code is nonzero if anything didn't make it, but the rest of the batch
still gets sent.

`del` processes each id independently, printing `deleted: <id>` as soon as
it's done. If one fails (even after retries), the rest of the batch still
goes ahead rather than aborting, with a summary and nonzero exit code at
the end if anything genuinely failed. A "not found" result is genuinely
ambiguous between "already deleted" (common if you re-run `del` on the
same id) and "never existed" — a deleted document returns the same `404`
as one that never was — `del` won't overclaim which.

Add `-D`/`--debug` to any command to see the raw JSON of every API call on
stderr.

## Version

```sh
my-boox --version
```

Prints the version and the git commit it's actually running from (with a
`-dirty` suffix if the working tree has uncommitted changes). Prefers the
live commit from the repo on disk; falls back to a baked-in placeholder
only if this copy was moved somewhere without its `.git` directory.

Current release: **v5.2**.

## Type checking

`pyright` in strict mode, clean:

```sh
pyright            # 0 errors, 0 warnings, 0 informations
```

`my-boox.py` is a symlink to the extensionless `my-boox`, because pyright
only analyses `.py` files.

Four inference-only rules (`reportUnknown{Variable,Member,Argument,Lambda}Type`)
are relaxed in `pyrightconfig.json`. Everything this script handles is JSON
from an API that makes no guarantees about its own fields — sizes and
timestamps arrive as numbers *or* strings, which is exactly the crash that
`_as_int` exists for — so modelling the payloads as `TypedDict`s would encode
promises the server does not keep. Every rule that catches a real defect
(unbound locals, `Optional` access, bad arguments, redeclarations) stays at
strict's default and reports zero.

## Exit codes

| Code | Meaning |
| --- | --- |
| 201 | wrong usage / missing arg / unknown command / refused to overwrite an existing file |
| 202 | `send`: every failure was a locally missing file (if anything else failed too, 204 wins) |
| 203 | a dependency (`requests`, `oss2`) is missing — the managed venv is absent or half-built |
| 204 | action failed even after re-auth retry, or a file failed to upload |
| 205 | re-auth step itself failed (no email configured, no code entered, stdin not a terminal, or the request-code/obtain-token call failed) |
| 206 | no config file found, or the `--config PATH` given does not exist |
| 130 | interrupted with Ctrl-C |

## License

MIT — see [LICENSE](LICENSE). Retains the original copyright notice from
`hrw/onyx-send2boox`, whose logic this project is substantially based on
and was rewritten from.
