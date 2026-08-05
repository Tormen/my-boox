# my-boox

Send, list, and delete files on an Onyx BOOX e-ink device via BOOXDrop —
from the command line.

A single self-contained Python script. No manual dependency installation:
on first run it creates its own virtualenv (under `~/.python.venv/my-boox`)
and installs everything it needs into it, then re-execs itself under that
interpreter.

## Origin

This started as a shell wrapper around
[hrw/onyx-send2boox](https://github.com/hrw/onyx-send2boox) (MIT licensed,
© 2022 Marcin Juszkiewicz), then absorbed and rewrote that project's logic
directly (its `boox.py`, `send_file.py`, `obtain_token.py`,
`request_verification_code.py`, and `delete_files.py`) into one file, with:

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
  retries the action once — no more running three separate scripts by hand.
- **Actual device delivery, not just account registration.** `saveAndPush`
  alone only registers a file in the account's listing — confirmed by an
  early test where a file sat registered for a long time without ever
  reaching the device. Real delivery requires *also* writing the file's
  document directly to `/neocloud/` (a Couchbase Sync Gateway instance
  speaking a CouchDB-style replication protocol, which the device actually
  replicates against) as a self-contained write, independent of whatever
  `saveAndPush` does server-side. `send` does this manual write first,
  then calls `saveAndPush`.
  A cleaner-looking alternative was tried — instead of a separate document,
  bumping a new revision directly onto the document `saveAndPush`'s own
  `cbMsg` response says it already creates server-side, avoiding a second,
  orphaned document entirely. It failed on two separate real attempts with
  a `504 Gateway Timeout` on the immediate follow-up write, plausibly a
  race against `saveAndPush`'s own asynchronous backend replication (the
  self-contained approach never depends on anything `saveAndPush` just
  wrote, so it doesn't hit this). Reverted in favor of the confirmed-
  reliable version. Downside: creates a second, orphaned `/neocloud/`
  document with its own id, visible as a duplicate row in the BOOXDrop web
  UI (`ls` only shows the real one, via `push/message`) — `del` cleans up
  both.
- **`del` that actually works**, for anything sent by this tool. Using the
  manually-created document's id/rev (saved to a small local state file
  next to your config) *and* discovering `saveAndPush`'s own document (its
  id equals the api/1 id), `del` submits real CouchDB/Sync Gateway
  tombstones for both — not just the legacy `push/message/batchDelete` REST
  call, which returns success but was confirmed (by waiting well over a
  minute) to not actually remove anything on its own. Files sent before
  this feature existed, or via the website, aren't in the state file, so
  `del` on those only finds (at most) the `saveAndPush`-created document via
  discovery — see the note in Use below.
- **A central pacing/retry layer for every request.** Multi-file `send`
  batches were hitting `502`/`504` errors from Onyx's backend under
  back-to-back requests — initially fixed with `send`-specific retry
  logic, then generalized: every outgoing request in the whole tool (not
  just `send`, also `ls` and `del`) now routes through one function that
  paces requests and retries transient failures with backoff, configurable
  via `api_pacing_seconds`/`api_retry_attempts`/`api_retry_delay_seconds`.
- **A separator line sized to your actual filenames**, not a fixed 57-dash
  string.
- **A fixed remote-filename bug**: the upstream code produced names like
  `uuid4()..pdf` (double dot) because it kept the leading dot from
  `os.path.splitext()` and added another one.

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

## Configure

```sh
my-boox --create-config              # print a template to stdout
my-boox --create-config ~/my-boox.ini # write it to a specific path
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
duplicate_check_on_send = true
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
- `duplicate_check_on_send` — `true`/`false`. When `true` (default),
  `send` skips any file whose name and size already match something on
  your account. Set `false` to always upload regardless.

### Config file search order

`my-boox` looks for `my-boox.ini` in, in this order:

1. `/LINKS/default/`
2. `~/`
3. `/etc/`
4. `/usr/local/etc/`

First match wins. Check where it's currently resolving to:

```sh
my-boox --config
```

Or point at a specific file for one run:

```sh
my-boox --config /path/to/other.ini ls
```

## Use

```sh
my-boox send some-document.pdf
my-boox send a.pdf b.pdf c.pdf        # multiple files in one call
my-boox ls
my-boox del 6835ffbf89ac7242e1ada708
my-boox del 6835ffbf89ac7242e1ada708 6a6df289d2c1f36e5ee55b9d
```

With multiple files, `send` keeps going even if one fails (missing file, transient
server error, etc.) — failures are summarized at the end and the exit code is
nonzero if anything didn't make it, but the rest of the batch still gets sent.

Every request any command makes — `send`, `ls`, and `del` alike, including
each of the several requests a single `send` involves under the hood — goes
through the same central pacing/retry layer: a minimum interval between
requests, and automatic retries with backoff on `502`/`503`/`504`,
connection errors, or timeouts (which Onyx's own web client also hits
occasionally). Both are configurable — see `api_pacing_seconds`,
`api_retry_attempts`, and `api_retry_delay_seconds` under Configure above.

`send` also skips any file whose name **and** size already match something
already on your account — checked once per batch (plus against files already
sent earlier in the same batch), so re-running the same command, or including
the same file twice, doesn't create duplicate uploads. Set
`duplicate_check_on_send = false` to turn this off.

> **`del` fully works for anything sent by `my-boox` itself** (tracked via
> a small `<config>.neocloud-state.json` file created alongside your config).
> Each id is processed independently — printed as `deleted: <id>` as soon
> as it's done, and if one fails (even after retries), the rest of the
> batch still goes ahead rather than aborting, with a summary and nonzero
> exit code at the end if anything genuinely failed.
> For files sent before this feature existed, or via the website, `del`
> still only runs the legacy `push/message/batchDelete` call, which is
> confirmed to *not* reliably remove the file from the device — use the
> BOOXDrop web UI's trash icon for those instead. `del` tells you explicitly
> when an id falls into this untracked category.

Add `-D`/`--debug` to any command to see the raw JSON of every API call on
stderr.

The very first time you run any command with no token yet configured,
you'll be walked through re-authentication automatically:

```
$ my-boox ls
my-boox: auth failed, requesting a fresh verification code...
Code for token requested. Check email.
Check you@example.com for the 6-digit code, then enter it: 123456
        ID               |    Size    | Name
-------------------------|------------|-------------------------
6a6df289d2c1f36e5ee55b9d |         53 | colors.properties
```

## Version

```sh
my-boox --version
```

Prints the version and the git commit it's actually running from (with a
`-dirty` suffix if the working tree has uncommitted changes). Prefers the
live commit from the repo on disk; falls back to a baked-in placeholder
only if this copy was moved somewhere without its `.git` directory.

Current release: **v2.1**.

## Exit codes

| Code | Meaning |
|------|---------|
| 201  | wrong usage / missing arg / unknown command |
| 202  | file not found (`send`) |
| 203  | venv bootstrap failed |
| 204  | action failed even after re-auth retry |
| 205  | re-auth step itself failed (no email configured, no code entered, request-code or obtain-token call failed) |
| 206  | no config file found and none could be resolved |

## License

MIT — see [LICENSE](LICENSE). Retains the original copyright notice from
`hrw/onyx-send2boox`, whose logic this project is substantially based on
and was rewritten from.
