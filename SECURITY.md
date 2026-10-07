# Relay security review (v0.9.0)

Three independent reviews looked at Relay 0.8.1: the app page and its bridge, everything that talks to the network
(web reading, phone page, Telegram, sync, sign-ins), and everything that touches your files and computer (terminal,
files, documents, updates). The problems below were fixed in 0.9.0 and are covered by automated tests.

## Fixed: serious

| Problem | Risk | Fix |
|---|---|---|
| Terminal commands joined with a single `&` (e.g. `ls & open -a Calculator`) passed as an allow-listed command | A bot (or a web page it read) could run a second command without asking | The command splitter handles `&`, `|`, `;`, newlines; anything with shell tricks (`$()`, backticks, redirects, `-EncodedCommand`…) always asks |
| No Content-Security-Policy on the app page; some synced fields were shown without escaping | A bad sync file could run code inside Relay | Strict CSP (no inline scripts, nothing loaded from outside); all synced text is cleaned and escaped |
| Sync accepted unencrypted device files, and synced "loosening" settings (terminal allow-list, home-network access, "act without asking") | Anyone who can write to your cloud folder could change Relay's safety settings on all computers | Sync folders must have a password (scrypt + AES-256-GCM); every computer's file is authenticated with its id; safety settings stay on each computer; far-future timestamps are ignored; deleting several things at once asks first |
| Web reading could reach home-network devices through IPv6 forms of private addresses or a name that switches address (DNS rebinding) | A web page could make a bot read your router or NAS | Address checked at connect time, all IPv4-mapped / NAT64 / 6to4 forms blocked, 15 MB limit; Relay Browser blocks home-network requests too |
| The page could ask the main process to open any file or restore any path | A compromised page could start a program | Only Relay's own page may call the main process; programs and scripts are never opened (only shown in their folder); undo/restore only works on Relay's own records |

## Fixed: medium

- Phone page: the pairing key no longer goes into web addresses; it is traded for an HttpOnly, SameSite=Strict
  session cookie. Requests must use the address in the QR code (stops DNS-rebinding tricks), cross-site posts are
  refused, and wrong keys lock an address out for 15 minutes. The page's script is a separate file under a strict CSP.
- Telegram: pairing is cancelled after 5 wrong codes.
- Phone attachments can only come from Relay's inbox and output folders.
- Chart drawing: data can no longer break out of the chart script; document windows can't open other windows or
  navigate; temporary files use private folders.
- Speech model downloads can't write outside the models folder (Windows path trick).
- Sign-in callback page escapes everything and only accepts the sign-in it started.
- Window permissions: camera, location, notifications from pages, USB/serial/HID devices are refused; the
  microphone is allowed only for Relay's own page (voice).
- `create_document` asks before saving into one of your own folders.
- Saving an API key for a changed server address, or a connector with a new command, asks on the computer.
- Scheduled runs never run commands that delete or send data, and never send email.
- Backups restore through the same checks as synced data.
- Release builds turn off Electron's run-as-Node mode, `NODE_OPTIONS` and debugger flags (fuses), and only load the
  app from its package.
- Dependencies: the MCP SDK was updated (GHSA-6qxp-vccf-f47h); `npm audit` reports 0 issues in what ships.

## New in 0.9 and designed safe

- **Email**: app passwords are stored encrypted; reading never marks mail as read; sending always shows the full
  email first, can't be set to "always allow", never happens in a scheduled run, and can only attach files Relay
  made or you attached. Email text is treated as information, never as instructions. Servers must use TLS.
- **Updates**: only files listed in a release list signed with Relay's release key (Ed25519) and matching its
  SHA-256 are installed. The release key never ships with the app.

## Known limits (by design)

- Relay is ad-hoc signed, not notarized by Apple or signed with a Windows certificate, so macOS and Windows warn on
  first install.
- The phone page uses plain HTTP on your home network (phones don't trust self-made certificates). Use it at home,
  not on public Wi-Fi; use Telegram when away.
- Bots can read any public web page you allow them to; a page could try to trick a bot. Actions that change things
  (commands, clicks that buy/send, email) always ask you first.
