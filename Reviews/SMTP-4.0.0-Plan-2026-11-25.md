[parm]:title = 'SMTP 4.0.0: plan 2026-11-25'
[parm]:toc = 2 3


# SMTP 4.0.0: plan, 2026-11-25

Decisions made on 2026-11-25:

* SMTP gets STARTTLS in the next version. Port 587 is the standard submission port, and some providers (Postmark among them) offer nothing else.
* `Secure` takes the values 0, 1 and 2, where 2 means STARTTLS. A server that does not offer STARTTLS when it was asked for is an error, never a silent fallback.
* There is no shared account any more. The tests get a fake SMTP server plus an optional live test with private credentials; `SendEmailExample` takes the caller's own server and credentials.
* The release carrying these changes is 4.0.0.

Finding numbers (B2, S1, …) point to `Reviews/SMTP-Review-2026-11-25.md`.

## STARTTLS

### Interface

* `ConnectionParameters.Secure` accepts 0, 1 and 2. 0 means no encryption, 1 means TLS from the first byte (today's meaning, unchanged), 2 means STARTTLS.
* The default port follows `Secure` as long as the user has not set `Port` explicitly: 465 for 1, 587 for 2. 0 keeps today's behaviour; 25 is the conventional value.
* The help page gets a short table: which value goes with which port, and which providers need which.

Existing code keeps working: nobody can have set `Secure` to 2 so far.

### Sequence in `Connect`

For `Secure←2`:

1. `LDRC.Clt` without certificate arguments, as for `Secure←0`.
2. Read the greeting (`220`).
3. `EHLO`.
4. Check the EHLO response for a `STARTTLS` line. If it is not there, close the connection and return an error ("server does not offer STARTTLS"). No fallback to plain text: an attacker who can strip that line would otherwise get the password (review S2).
5. Send `STARTTLS`, expect `220`.
6. `LDRC.SetProp _clt 'StartTLS' (cert.AsArg,('SSLValidation' flags)('Address' server))`, as in the Conga User Guide, section 5.3.1.
7. `EHLO` again. RFC 3207 requires the client to discard everything learned before the upgrade, so `EHLOResponse` is replaced by the new answer. Some servers only offer `AUTH` after the upgrade.
8. `Logon` continues as today.

### What has to change along the way

* `EHLO` and `Do` must read multi-line responses properly: collect chunks until a line starting with three digits and a blank arrives, instead of assuming two pieces (review B17). STARTTLS makes this more pressing, because the EHLO response is parsed twice.
* `Logon` must not reject a server whose EHLO response has no `AUTH` line when no password is set (review B5).
* `PerformChecks` must stop tying the password to `Secure` the way it does now (review S2). Proposed rule: a password with `Secure←0` is an error unless the user sets an explicit flag such as `AllowPlainTextAuth←1`.
* The `TLSFlags` setter must accept combinations of flags (review B14), and the default should validate the server certificate (review S1).

### Testing

The fake SMTP server (next section) can do STARTTLS too: Conga can upgrade a server connection the same way (`SetProp con 'StartTLS'`, Conga User Guide section 5.4.1), using the test certificates Dyalog ships in `[DYALOG]/TestCertificates`. That allows tests for the full sequence, for a server that does not offer STARTTLS, and for `EHLO` arriving in several pieces, all without network.

The optional live test against a real provider on port 587 completes it.

### Estimate

STARTTLS itself is 40 to 60 lines in `Connect` and `ConnectionParameters`. The multi-line response handling in `Do` is the risky part, and the fake server is a small project of its own. Roughly: one day for STARTTLS plus response handling, one to two days for the fake server and the tests built on it.

## Replacing the shared Gmail account

### Tests

* A fake SMTP server, built on Conga's `Srv`, running in a thread of the test workspace. It answers from a script (greeting, EHLO lines, AUTH challenges, 250/354/221), records everything it receives, and can be told to misbehave: split responses, refuse STARTTLS, reject AUTH, time out.
* Most of the review's test findings become offline tests on top of it, or need no server at all because they only check the composed message text: dot-stuffing (B2), `X-Mailer` (B7), header encoding (B9), closing boundary (B10), header folding (B12), `AUTH PLAIN` (B4), no `AUTH` (B5).
* One optional live test sends a real message. It reads server, user and password from outside the repository (an environment variable or a file that is not committed) and is skipped, not failed, when none are there.
* The test suite then runs in batch. The questions asking a person to confirm receipt go, except for the optional live test.
* The `USER` section in `cider.config` loses its credentials.

### `SendEmailExample`

* It takes the caller's server and credentials. Proposed syntax: `r←connectionParms SendEmailExample emailAddress`, where `connectionParms` is an instance of `ConnectionParameters`. The optional attachments, today the left argument, move to the right: `connectionParms SendEmailExample emailAddress filenames`. (A tradfn's left argument cannot be split in the header, so they cannot both go left.)
* The hard-coded Gmail credentials are removed.
* The help page and the README explain how to get credentials: Postmark as one suggestion, a provider offering port 465 as another (works with today's SMTP), a Gmail account with an app password as a third, each with the matching `Secure` and `Port`. They also say not to commit the credentials (review S3).
* The "works out of the box" statements go.

## Release 4.0.0

Breaking, because `SendEmailExample` changes its syntax, and because of the stricter security defaults below.

Belongs in the same release, since the code is touched anyway or the change is breaking too:

* STARTTLS (`Secure←2`), the port default, multi-line responses (B17).
* Servers without authentication (B5), and refusing a password over plain text unless explicitly allowed (S2).
* Certificate validation by default and combinable `TLSFlags` (S1, B14). This can break users talking to a server with a self-signed certificate; the help needs to say how to switch validation off.
* The new tests and the removal of the shared account (B1, S3), `SendEmailExample` (B6).

Independent of the release number, and worth fixing first because they are plain bugs with small fixes: dot-stuffing (B2, S4), `now` (B3), `AUTH PLAIN` (B4), `X-Mailer` (B7). They could go out as 3.1.3 before 4.0.0 if that is wanted.
