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
- **Actual device delivery, not just account registration.** `saveAndPush`'s
  response includes a `cbMsg` field — confirmed, by capturing a real browser
  session, to be the id/rev of the document it writes server-side to
  `/neocloud/` (a Couchbase Sync Gateway instance speaking a CouchDB-style
  replication protocol), which is what the device actually replicates
  against. An earlier version of this tool additionally wrote its own
  separate `/neocloud/` document before calling `saveAndPush`, which turned
  out to be redundant — it created an orphaned duplicate visible in the
  BOOXDrop web UI (but not in `ls`, since `push/message` only ever knew
  about the real one). Removed; `send` now just calls `saveAndPush` and
  captures the `cbMsg` id/rev it returns.
- **`del` that actually works**, for anything sent by this tool. Using the
  `cbMsg` id/rev captured at send time (saved to a small local state file
  next to your config), `del` submits a real CouchDB/Sync Gateway tombstone
  to `/neocloud/` — not just the legacy `push/message/batchDelete` REST call,
  which returns success but was confirmed (by waiting well over a minute)
  to not actually remove anything from the device on its own. Files sent
  before this feature existed, or via the website, aren't in that state
  file, so `del` on those still only does the legacy call and may not
  actually clear them from the device — see the note in Use below.
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
```

Fill in `email` with the address tied to your Onyx account. Leave `token`
blank — it gets filled in automatically the first time you run a command
and go through the re-auth prompt. `cloud` is your BOOXDrop server region
(`eur.boox.com` for EU, `push.boox.com` for US/VN).

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
my-boox ls
my-boox del 6835ffbf89ac7242e1ada708
my-boox del 6835ffbf89ac7242e1ada708 6a6df289d2c1f36e5ee55b9d
```

> **`del` fully works for anything sent by `my-boox` itself** (tracked via
> a small `<config>.neocloud-state.json` file created alongside your config).
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
