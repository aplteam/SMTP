[parm]:title = 'SMTP review 2026-11-25'
[parm]:toc = 2 3


# SMTP review, 2026-11-25

Scope: the package source (`APLSource/SMTP`), the API covers, `SMTP_Help.md`, `README.md`, `apl-package.json` and, more lightly, the test suite. `Admin.Make` was left out. State reviewed: branch `dev` at `bbf4821`, package version `3.1.2+26`.

Nothing was changed. Each finding says how it was established:

* **Checked**: reproduced in Dyalog 21.0 with `-script`, using the real class files from `APLSource/SMTP`. No network, no mail server.
* **Reading**: follows from the code, but was not executed, usually because it needs a live server or a particular machine setup.

Line numbers are APL line numbers: the header of a function is line 0, and for a function inside a class they count from that function's own header.

The Gmail account used by `SendEmailExample` and the tests is treated as dead (Kai, 2026-11-25), which makes it the first finding.

## Summary

The SMTP dialogue works for the case it has been tested with: Gmail, port 465, `AUTH LOGIN`, ASCII headers, short recipient lists. Outside that case it fails in many places. The most serious problems:

1. The account that makes the package "work out of the box" no longer exists, so the example, the tests and the README all fail today.
2. The body is not dot-stuffed. A line consisting of a single `.` ends the message early, and whatever follows is sent to the server as SMTP commands.
3. On a Windows machine without daylight saving time, every `Send` fails with an `INDEX ERROR` while building the `Date` header.
4. The `AUTH PLAIN` branch cannot run at all (`SYNTAX ERROR`), and servers that need no authentication are refused even though the comments say they are supported.
5. Server certificates are not validated by default, so the password can be read by anybody in the middle.

The tests cannot catch most of this: every test that sends mail needs the live account and a person to confirm receipt, and nothing tests the composed message offline.

## Bugs

### B1: the shared Gmail account is gone

`SendEmailExample[12-15]` and `cider.config` (section `USER`) use `smtp.tatin.tests1@gmail.com` with a hard-coded app password. With the account cancelled:

* `SendEmailExample` fails for everybody.
* Every `Test_SendEmail_*` test fails, so the suite cannot pass.
* `README.md` and `SMTP_Help.md` promise something that does not work ("works out of the box", "ready-to-go example").

There will be no shared account in its place: the tests get a fake SMTP server plus an optional live test that reads private credentials from outside the repository, and `SendEmailExample` takes the caller's own connection parameters. A public credential of the same kind is not an option with a service like Postmark anyway, see S3. (Reading; Kai's information.)

### B2: the body is not dot-stuffed

`Message.Compose[33]` joins the body lines with CRLF and nothing else. `Connection.Send[34]` then sends `text,CRLF,'.'`. A body line that is exactly `.` therefore ends the DATA phase early; the server delivers a truncated message and treats every following line as an SMTP command. A line that merely starts with a dot loses that dot at the receiver.

The function `chunk` does dot-stuffing, but it is only applied to base64-encoded attachments, which cannot contain dots.

Checked: a body of `'Line one' '.' 'last line'` produces a line holding a single `.` in the composed text. This is also a security hole, see S4.

### B3: `Message.now` fails without daylight saving time, and is wrong for half-hour zones

`now[19]` indexes `x[(5 8)[⊃x]]`, where `⊃x` is the result of `GetTimeZoneInformation`. Windows returns 0 (`TIME_ZONE_ID_UNKNOWN`) when the time zone has no daylight saving time or automatic adjustment is switched off. With `⎕IO←1` that is an `INDEX ERROR`, and since `Compose` builds the `Date` header first, every `Send` fails on such a machine.

`now[20]` formats the offset as `100×minutes÷60`, which is only right for whole hours. India (UTC+05:30) gets `+0550`.

Checked: the expression from `now`, fed with the values Windows reports for India, gives `+0550` for a return code of 1 and `INDEX ERROR` for 0. Not run on a machine actually configured that way.

### B4: `AUTH PLAIN` cannot work

`Connection.Logon[21]` reads `(⊃⊃⎕CLASS ⎕THIS)##.Message.Base64Encoding …`. Every other line has `.##`; without the dot the line is a `SYNTAX ERROR`. Any server that offers `PLAIN` but not `LOGIN` cannot be used. Gmail offers both and the `LOGIN` branch comes first, which is why it never showed.

Checked with the same construct in a namespace: `(⎕THIS)##.M.Enc 'abc'` gives `SYNTAX ERROR`, `(⎕THIS).##.M.Enc 'abc'` works.

### B5: servers without authentication are refused

The leading comments of `Logon` say: "If no password is set, then authentication is not done". The code does not check the password at all. `Logon[11-12]` returns an error when the EHLO response carries no `AUTH` line, and `Send` always calls `Logon`. A local relay or an internal mail server that accepts mail without authentication therefore cannot be used. (Reading.)

### B6: `SendEmailExample` throws away the error message

`SendEmailExample[33]` builds the failure text with `¯1↓(⊆msg),¨⎕UCS 13`. The `¯1↓` drops the last *item* of the nested vector, not the trailing CR, so with a simple `msg` the whole message disappears. Checked: `rc←1 ⋄ msg←'535 Authentication failed'` gives `Failed with rc=1; ` and nothing else. This is the same kind of loss commit `c274f88` fixed elsewhere.

### B7: `X-Mailer` is `APLTeam SMTP Client .`

`ConnectionParameters.make[11]` takes `2⊃` of `Version`. Since 3.0.0 `Version` returns a simple character vector such as `3.1.2+26`, so `2⊃` picks the character `.`. Without Tatin `Version` returns `''`, the `INDEX ERROR` is trapped, and the version is simply missing. Checked: with `Version` returning `'3.1.2+26'` the property reads `APLTeam SMTP Client .`.

### B8: an address without a dot in the domain crashes `Send`

`MailParameters` accepts `kai@localhost` (it checks for an `@` and a length above 5). `Message.extractAddr` requires a dot after the `@` and returns `''` for it, and `normalizeAddr` then fails with `DOMAIN ERROR` on `⊣/''`. Checked with both functions. The same should happen for anything else `extractAddr` rejects; an internal host name is the realistic case.

### B9: non-ASCII header values are encoded wrongly

`Compose[8]` (the local `addHeader`) wraps a non-ASCII value as `=?utf-8?B?…?=` but passes the raw characters to `Base64Encoding`, which encodes one byte per character. Characters from U+0080 to U+00FF come out as Latin-1 bytes labelled UTF-8; anything above U+00FF is silently truncated to its low byte. Checked: `Base64Encoding 'Jäger'` gives `SuRnZXI=` (Latin-1), and `'Ł'` gives `QQ==`, which decodes to `A`.

The subject is converted to UTF-8 first (`Compose[18]`) and is fine. Affected are `Organization`, the display name in `From`, and `Reply-To`.

### B10: the multipart message has no closing boundary

`Compose[38]` and `Compose[45]` end every part with `--boundary`, including the last one, which should be `--boundary--`. Most mail clients cope, but the message is not valid MIME. Checked in the composed text.

### B11: a missing attachment file gives `RANK ERROR` instead of an error message

When the file behind an attachment disappears between `AddAttachment` and `Send`, `ProcessAttachment` returns `''` as intended, and `Compose[41]` then builds its message with `1⊃i⊃Attachments`. The item is an `Attachment` instance, a scalar, so `1⊃` is a `RANK ERROR`. Checked; the error is reported at `Compose[41]`.

### B12: recipient headers are not folded, and an empty `CC:` header is sent

`CompileRecipients` says it "splits at commas when the result exceeds (a multiple of) 78", but no line break is ever inserted: four CC addresses of 50 characters each produce a single 210-character header line. RFC 5322 recommends 78 and requires at most 998 characters, so a long CC list can get the message rejected. Without any CC, `Compose[17]` writes a header line consisting only of `CC:`. Both checked.

### B13: `Domain` is computed once and then sticks

The getter of `ConnectionParameters.Domain` stores the domain derived from `Userid` the first time it is read. `⎕NEW Connection` reads it (via `PerformChecks`), so changing `Userid` afterwards keeps the old domain. Checked: after `Userid←'a@first.com'`, reading `Domain`, then `Userid←'b@second.org'`, `Domain` still says `first.com`.

### B14: `TLSFlags` refuses combined flags

The Conga certificate flags are bit flags meant to be added together, but `TLSFlags[3]` (setter) only accepts a single power of 2. Checked: `40` (8+32) is rejected. That also stops anybody from setting a sensible combination for S1.

### B15: a `CongaRef` given as a name never works

`ResolveCongaRef` is dyadic, but `ResolveCongaRef[26]` calls it monadically for the case where `CongaRef` is a character vector such as `'#.Conga'`. The `VALUE ERROR` on `rootname` is swallowed by the surrounding `:Trap 0`, so the user gets "does not point to a valid instance of Conga" for a perfectly valid name. (Reading.)

### B16: `Connect` ignores whether Conga could be initialised

`Connect[7]` calls `Init` and never looks at the result. If Conga cannot be found, `LDRC` stays `''` and the next line fails with an untrapped error instead of returning `rc msg`. On top of that `Init[53]` sets `rc←¯1×LDRC≢''`, which is `¯1` on success and `0` on failure, the reverse of every other return code in the package. (Reading.)

### B17: the EHLO response is assumed to arrive in at most two pieces

`EHLO[5]` reads one more chunk if the first does not end with `250 `, and says itself that it "makes the assumption that the EHLO response is in 2 parts only". If a server sends it in three TCP packets, the `AUTH` line may be missing from `EHLOResponse` and `Logon` fails (B5's error message). `Do` has the same weakness for every multi-line response. (Reading.)

### B18: the end of DATA may time out with large attachments

`Do[76]` waits 3 × 2 seconds. After the final `.` a server may take longer than that to accept a large message (virus scanning, for example), and `Send` then reports `555 Conga timeout` and sends `RSET`, although the message may well be delivered. (Reading; not observed.)

### B19: `Message.Send` cannot work

`Message.Send` calls `Client.Send ⎕THIS`, but `Connection.Send` expects a `MailParameters` instance and immediately calls `PerformChecks`, which `Message` does not have. The method is public and exposed through the API cover `Message`. (Reading.)

### B20: smaller bugs

* HTML detection in `Compose` only looks at the first item of `Body`. An HTML body whose first line is `<!DOCTYPE html>` or blank is sent as `text/plain`. (Reading.)
* `GetAllMimeTypes` has no `.jpg` (only `.jpeg`), so JPG files go out as `application/octet-stream`; `.7z` maps to `application/x-7z-compresses` (should be `-compressed`). Both checked.
* `ProcessAttachment[9]` declares `charset="US-ASCII"` on every attachment, including PDFs and images. (Reading.)
* No `Message-ID` header is generated (checked in the composed text). Large providers count that against a message; it is one more reason why mail from this package lands in spam.
* `Send[26]` sends `MAIL FROM: <…>` and `RCPT TO: <…>` with a space after the colon. RFC 5321 has no space there; Gmail tolerates it, strict servers answer `501`. (Reading.)
* An `AUTH` offer without `LOGIN` or `PLAIN` makes `Logon[23]` signal `DOMAIN ERROR` instead of returning `rc msg`. (Reading.)
* `Send[10]` writes `From` into the caller's `MailParameters` instance when it is empty, a side effect on the caller's object. (Reading.)
* `Subject` and `Body` refuse characters above U+FFFF (`⎕DR` 320, emoji); `Org`, `Userid`, `Server`, `ReplyTo` and friends refuse anything above U+00FF (`⎕DR` 160). Both checked.
* The `Date` header ends in `(GMT)` whatever the offset, `+0100 (GMT)` for example. Harmless but wrong. Checked.

## Security

### S1: server certificates are not validated by default

`TLSFlags` defaults to 32, "accept server certificate without validating". So the connection is encrypted but not authenticated: anybody able to intercept the traffic can present their own certificate and read the password in clear (it is only base64-encoded). The default should validate; turning validation off should be the exception the user asks for. B14 currently gets in the way of setting a proper combination. (Reading.)

### S2: the password is sent unencrypted when `Secure` is 0

With `Secure←0` and a password, `Logon` happily sends `AUTH LOGIN` over a plain connection. `PerformChecks` even ties the two the other way round: it requires a password only when `Secure` is 1. There is no STARTTLS (see S5), so a connection on port 587 or 25 is either refused or unencrypted. (Reading.)

### S3: credentials in the repository and in the shipped code

The Gmail app password sits in `cider.config` (committed) and in `SendEmailExample`, which is part of the package. Publishing it was a deliberate decision: the account existed only for this purpose and was used for nothing else, so the damage was limited to losing the account, which is what happened. That reasoning does not carry over to whatever replaces it. With a service like Postmark a public token would let anybody send in the domain's name and get the account and the domain blocked, taking every other use of that domain down with it. Tests therefore need a way to get their credentials from outside the repository (environment variable, an uncommitted file). The help page's advice to "copy this password into the INI file" also deserves a sentence telling readers who use their own account not to commit it. (Reading.)

### S4: SMTP command and header injection

* Through the body: because of B2, a body line `.` followed by `MAIL FROM:…`, `RCPT TO:…`, `DATA` makes the server accept a second message chosen by whoever controls the body text. Crash reports, a typical use of this package, often contain user-supplied data. (Reading, follows from the checked B2.)
* Through headers: `Subject`, `To`, `CC`, `BCC`, `From`, `Org` and `ReplyTo` accept CR and LF. A subject such as `'Hi',CRLF,'Bcc: x@y.com'` adds a header; an address with an embedded CRLF ends up in a `RCPT TO:` line and becomes a command. (Reading.)

### S5: no STARTTLS

`Connection` only knows TLS from the first byte (port 465) or no encryption at all. Port 587 with STARTTLS is the standard submission port, and some providers (Postmark among them) offer nothing else. Conga supports it (`DRC.SetProp clt 'StartTLS' …`, Conga User Guide section 5.3.1). Design and estimate: `Reviews/SMTP-4.0.0-Plan-2026-11-25.md`.

## Documentation

### D1: out-of-date and misleading content

* `README.md` and the overview in `SMTP_Help.md` promise a working example (B1).
* `SMTP_Help.md` contains two overlapping Gmail recipes ("Using smtp.google.com yourself" and "Create your own GMail address for SMTP"), both describing a Google user interface that has changed since. The first ends with "copy this password into the INI file of your Tatin server in the `[EMAIL]` section", which is Tatin-specific. The note "valid as of 2021-09" contradicts the aside "not available anymore since March 2022" right above it.
* The host name is given as `https://smtp.google.com` (Preconditions) and `smtp.google.com` (Reference, Server). Gmail's SMTP server is `smtp.gmail.com`, and it is not a URL.
* The README calls the app password an "API key".

### D2: the reference is incomplete

* The result of `Connection.Send`, `(rc msg log)`, is not documented anywhere: that `rc` is `¯1` for a parameter problem, `1` for an SMTP error and `0` for success, what `msg` holds, and what `log` collects.
* The `Connection` class has no reference section at all (`Send`, `Connect`, `Logon`, `Logoff`, `Ping`, `Config`, `CongaRef`, `EHLOResponse`), nor do `Attachment`, `Version`, `History` and `Help`.
* Nothing says which platforms are supported (`apl-package.json` has `os_lin` and `os_mac` set to 0 while the code has Unix branches, see Q1) or that port 587 / STARTTLS is not supported.
* `Domain` is described as "Fully qualified domain name for logging on with `Userid` and `Password`" (help) and "URL of the SMTP server" (property comment). It is the client's domain sent with `EHLO`, defaulting to the part of `Userid` after the `@`.
* `TLSFlags` says what 32 means but not that it is insecure (S1).
* `Subject` and `Body` are documented as "UTF-8 text vector". They take ordinary Unicode characters; passing UTF-8 bytes would encode them twice.
* `apl-package.json` has `documentation: ""`.

### D3: documentation that says the wrong thing

* "Userid … defaults to `From`" (help and `ConnectionParameters.make[5]`): it is the other way round, `From` defaults to `Userid` (`Send[10]`).
* `XMailer` is documented as "Dyalog SMTP Client {versionNumber}"; the code says `APLTeam SMTP Client` (and B7).
* The comment of property `Userid` reads "Simple character vector that is going to be the subject line of the email", and so does the one of `Body`. Both are copied from `Subject`.
* The port comment in `ConnectionParameters.make[7]` says "578 otherwise"; the submission port is 587.
* `Help[1]` says it "prints an `]Open` statement"; it executes one.
* The comments of `Connection.Do` sit after `→go`, so they are not leading comments and tools such as ADOC do not show them. They refer to `smtp_stuff`, and `chunk` refers to `pop3_unstuff`; neither exists. `chunk` calls its result `rc`, but it is `r`.
* `ResolveCongaRef` talks about HttpCommand and carries a `⍝!!! testing only - remove :Access after testing` note.
* `CompileRecipients[3]`: the example in the comment does not match the function's syntax.
* The `Secure` table in the help has no header row and does not render as a table.

### D4: typos

* `SMTP_Help.md`: "youself", "machanism", "ative", "is a requirements", "on the left site", "You will be prompted for you password", "the `ConnectionParms` class" (should be `ConnectionParameters`), "In case no instance of the `AddAttachment`" (should be `Attachment`), "Setting" / "See all setting".
* `README.md`: "an email send via this sender address".
* `SendEmailExample`: "confidental", "uabuses", "filesnames".
* `History`: "ocurred", "construcctor", "remaind", "according to the changed Google introduced".
* `MailParameters`: the error messages call `@` an "ampersand" (CC, BCC, From); the comment of `From` reads "<john Doe)" and "you mail".
* `CompileRecipients[5]`: "tither", and a tradfn error message talking about `⍺`.

## Code quality

* Inline I-beam: `chunk[17]` uses `900⌶⍬` to find out whether the left argument was given. `0=⎕NC'len'` does the same without an I-beam, as `Xmit` already does.
* `:Trap 0`: `Connected[4]`, `unmake`, `ProcessAttachment[4]`, `Init` (the copy loop) and `ResolveCongaRef` all catch everything. B15 is an example of a real bug hidden this way. None of the handlers keeps `⎕DMX`, so the reason is lost in every case; `ProcessAttachment` in particular turns any error into "Error processing attachment" (and then B11).
* Dead code: the fields `Message.To` and `Message.Headers`, `Connection._conx` (and the property `Conx`, which is therefore always `''`), the locals `email` and `dom` in `Logon`, `mat` and `row` in `GetAllMimeTypes`, the left argument of `ProcessAttachment`, and `Message.Send` (B19).
* Design: `ReplyTo` and `Org` are properties of `ConnectionParameters`, although they belong to a message.
* Duplication: the same `Assert` is defined in four classes, with the odd `1↓(⊃∊⍵),11` that always comes out as 11.

## Tests

* No offline tests of the message text. Every send test needs the live account and asks a person to confirm receipt, so they cannot run in batch, and with B1 they cannot pass at all. Nothing tests `Compose` directly, which is why B2, B7, B9, B10 and B12 went unnoticed. Most of these can be tested by composing a message and checking the text, without any network.
* No tests of the protocol paths. `AUTH PLAIN` (B4), servers without `AUTH` (B5), EHLO in several pieces (B17) and dot-stuffing on the wire would need a server. A small fake SMTP server built on Conga's `Srv` in the same workspace would cover them without touching the internet.
* Not covered: the failure branch of `SendEmailExample` (B6), `now` (B3), `extractAddr` and `normalizeAddr` with unusual addresses (B8), a missing attachment file (B11).
* Credentials: the tests read the Gmail password from `cider.config`, which is committed (S3).
* `TestResults/` (code coverage output) is committed; it is generated and will change with every run.

## Open questions

* Q1: `apl-package.json` declares Windows only (`os_lin: 0`, `os_mac: 0`), yet the code has Unix branches (`now` calls `date -R`) and Conga runs everywhere. Is the restriction still needed, or a leftover?
* Q2: `History` lists 3.1.3 as "from 2026-08-??" while `apl-package.json` still says `3.1.2+26`. One of them needs adjusting before the next release.
