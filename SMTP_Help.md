[parm]:numberHeaders     = 2 3 4 5 6
[parm]:leanpubExtensions = 1



# `SMTP` Help!

## Overview

`SMTP` sends emails from APL code through an SMTP server of your choice: plain text or HTML, with or without attachments, with any Unicode characters in subject and body.

You need an account with an SMTP server (also called a mail submission server) to send email. `SMTP` does not come with one: the shared Gmail account that earlier versions used for `SendEmailExample` and the test cases no longer exists.

## User Guide

### Preconditions

You need:

* The host name of an SMTP server, for example `smtp.gmail.com`. That is a host name, not a URL.
* A user ID and a password for that server. Many providers do not accept your normal password here but ask you to create an "app password" for it.
* A server that offers either SSL/TLS from the first byte, usually on port 465, or an unencrypted connection, usually on port 25.

`SMTP` does not support STARTTLS yet, the mechanism that switches an unencrypted connection to an encrypted one, usually on port 587. Providers that offer nothing else, Postmark for example, cannot be used with this version.

The package is declared for Windows only.

### Where to get credentials

Any provider that offers SMTP on port 465 works. Gmail is one: with 2-Step Verification switched on for a Google account, you can create an app password for it in the account's security settings, and use `smtp.gmail.com` on port 465 with the email address as `Userid` and the app password as `Password`. Google changes its user interface from time to time, so search Google's help for "app passwords" when the settings look different.

Whatever account you use: **do not put its password into a file that is under version control**, and do not hard-code it into code you distribute. Anybody who finds it can send email in your name, and the provider is likely to block the account.

### Send a test email

`SendEmailExample` sends a test email with your own credentials:

```
      cp←⎕NEW SMTP.ConnectionParameters
      cp.Server←'smtp.gmail.com'
      cp.Userid←'me@gmail.com'
      cp.Password←'abcdefghijklmnop'
      cp SMTP.SendEmailExample 'john.doe@whatever.com'
Email was sent successfully
```

To attach files, add their names after the email address:

```
      cp SMTP.SendEmailExample 'john.doe@whatever.com' 'my.txt' 'foo.htm'
```

The email is sent from `Userid`. When something goes wrong the result says so and carries the server's answer, for example:

```
Failed with rc=1; 535 5.7.8 Authentication credentials invalid
```

### Usage

Steps to send an email:

1. Create an instance of the `ConnectionParameters` class.

2. Set at least `Server`, `Userid` and `Password`, and `Port` and `Secure` if the defaults (465 and 1) are not right for your server.

3. Create an instance of the `Connection` class by providing the instance of the `ConnectionParameters` class.

4. Create an instance of the `MailParameters` class.

5. Set at least `To`, `Subject` and `Body`.

6. Optionally add attachments.

   This can be done by calling the `AddAttachment` method of the instance of the `MailParameters` class. In case no instance of the `Attachment` class is passed as argument to `AddAttachment` it will internally create one from the argument.

   `AddAttachment` accepts three different arguments:

   * The attachment's filename. The `Attachment` class works out the MIME type from the extension.

   * The attachment's filename and its MIME type as two text vectors. Use this in case there is no extension, or an unusual one.

   * An instance of the `Attachment` class.

7. Call the `Send` method of the instance of the `Connection` class and pass the instance of the `MailParameters` class as an argument. This method will send the email; see "The result of `Send`" below.

Example:

```
      cp←⎕NEW SMTP.ConnectionParameters
      cp.(Server Userid Password)←'smtp.gmail.com' 'me@gmail.com' 'abcdefghijklmnop'
      myConnection←⎕NEW SMTP.Connection(,⊂cp)
      mp←⎕NEW SMTP.MailParameters
      mp.To←'john.doe@whatever.com'
      mp.Subject←'Crash report'
      mp.Body←'The application crashed.' 'See the attached report.'
      mp.AddAttachment 'C:\Temp\CrashReport.html'
      (rc msg log)←myConnection.Send mp
```

Notes:

* When properties of the classes `ConnectionParameters`, `MailParameters` and `Attachment` are set or changed they are checked straight away, so you are told immediately when something is wrong.

* Both classes `ConnectionParameters` and `MailParameters` offer a method `PerformChecks` which makes sure that all required properties have reasonable values, and that the setting of the properties is consistent.

  You don't have to call these methods because `SMTP` will do this anyway, but you may if you wish.

* A connection stays open after `Send`, so you can send several emails through the same instance of `Connection`. It is closed when the instance is deleted, or when you call `Logoff`. Servers close a connection that has not been used for a few minutes; the next `Send` then connects and logs on again.

* Line breaks (CR or LF) are refused in every property that ends up in a header line or an SMTP command (`To`, `CC`, `BCC`, `From`, `Subject`, `Server`, `Userid`, `Domain`, `Org`, `ReplyTo`, `XMailer`). Otherwise text from an untrusted source could add headers or recipients.

* Lines in the body that start with a dot are sent with an extra dot, as SMTP requires; the receiving server removes it again. You don't need to do anything about it.

### The result of `Send`

`Connection.Send` returns a vector of three items: `(rc msg log)`.

`rc` is one of:

| rc  | Meaning
|-----|---------
| 0   | The server accepted the email.
| ¯1  | Nothing was sent: a parameter is missing or invalid, an address is not valid, an attachment could not be read, no connection to the server could be established, or the server offers neither AUTH LOGIN nor AUTH PLAIN.
| 1   | The server refused something. A code of 555 means that `SMTP` itself ran into a problem during the dialogue with the server, for example a timeout.

`msg` is empty for success. Otherwise it says what went wrong; for `rc=1` it is the server's answer, starting with the three-digit SMTP code.

`log` holds the server's answers to the `RCPT TO` commands, one per recipient (`To`, `CC` and `BCC`), plus the answer to `RSET` when sending failed after the first recipient.

## Reference

### The "ConnectionParameters" class

An instance of this class must be passed as argument to `⎕NEW Connection`.

#### Server

Host name or IP address of the SMTP server, for example `smtp.gmail.com`.

#### Password

The password for `Userid`. Required when `Secure` is 1. It may contain any Unicode character; it is sent as UTF-8.

With `Secure` 0 and no password, `SMTP` does not authenticate at all. That is what a relay in an internal network usually expects.

#### Userid

The user ID for logging on to the server, often an email address. Required when `Secure` is 1. An empty `MailParameters.From` defaults to `Userid`.

#### Secure

| Value | Meaning
|-------|---------
| 1     | SSL/TLS from the first byte (default)
| 0     | Not encrypted

With 0 the password is sent without encryption (base64-encoded only), so use 0 only for a server in a network you trust.

#### Port

Defaults to 465, the usual port for SSL/TLS. An unencrypted server usually listens on port 25.

#### Domain

The client's domain, sent to the server with the `EHLO` command. As long as it is not set, it is the part of `Userid` after the @, and it follows later changes of `Userid`. Without a `Userid`, set `Domain` yourself.

#### Org

Optional: the content of the "Organization" header. Any Unicode character is fine.

#### ReplyTo

Optional: the content of the "Reply-To" header: an email address, optionally with a display name, as described for `MailParameters`.

#### XMailer

The content of the "X-Mailer" header. Defaults to "APLTeam SMTP Client" followed by the version number of the package, if available.

#### TLSFlags

Certificate validation flags for Conga, see the Conga User Guide, Appendix C. The default is 32: accept the server certificate without validating it. The connection is then encrypted, but nothing makes sure that the other end really is the server you asked for, so anybody able to intercept the connection could read the password.

#### PerformChecks

A method that returns an empty vector when all required properties are set, and a message listing what is missing otherwise.

### The "MailParameters" class

An instance of this class must be passed as argument to the instance method `Send` of the `Connection` class.

An address in `To`, `CC`, `BCC` or `From` may carry a display name, for example `'John Doe <john.doe@whatever.com>'`. The display name may contain any Unicode character; it is encoded as required (RFC 2047). The email address itself must consist of ASCII characters: an address like `jürgen@example.com` would need the SMTPUTF8 extension, which `SMTP` does not support, so `Send` refuses it with `rc=¯1`. The domain part does not need a dot, so an internal address like `kai@localhost` is fine.

#### To

A simple text vector holding exactly one email address.

#### CC

A simple text vector holding one or more email addresses. Multiple addresses must be separated by commas.

#### BCC

A simple text vector holding one or more email addresses. Multiple addresses must be separated by commas.

#### From

A simple text vector holding an email address. If it is left empty it defaults to `Userid` of the `ConnectionParameters` instance. That is a good idea in most cases, because many servers accept only their own addresses as sender, and others are likely to put your mail into the spam folder.

#### Subject

A simple text vector. Any Unicode character is fine: pass ordinary characters, not UTF-8 bytes, which would be encoded twice.

#### Body

Either a simple text vector or a vector of simple text vectors (lines). As with `Subject`, any Unicode character is fine. The body is sent as UTF-8.

A body that contains `<html` in its first line is sent as HTML, otherwise as plain text.

#### Attachments

Empty or a vector of instances of the `Attachment` class, added by the `AddAttachment` method.

#### AddAttachment

A method that adds an attachment; see "Usage" above for the arguments it accepts.

#### PerformChecks

A method that returns an empty vector when `To`, `Subject` and `Body` are set, and a message listing what is missing otherwise.

### The "Connection" class

Create an instance with `⎕NEW Connection(,⊂connectionParameters)`. The constructor checks the parameters and signals an error if something required is missing.

#### Send

Takes an instance of `MailParameters`, sends the email and returns `(rc msg log)`; see "The result of `Send`". It connects and logs on first if necessary.

#### Connect, Logon, Logoff

`Send` calls `Connect` and `Logon` when needed, so you rarely need them. `Logoff` sends `QUIT` and closes the connection. All three return `(rc msg)`.

#### Ping, Reset

Send `NOOP` and `RSET` respectively and return `(rc msg)`.

#### Connected

Returns `(flag msg)`: whether the connection to the server is up.

#### LoggedOn

Read-only property: 1 after a successful logon.

#### EHLOResponse

Read-only property: the server's answer to `EHLO`, listing what the server supports.

#### ConnectionParms

Read-only property: the instance of `ConnectionParameters` the connection was created with.

#### CongaRef

Shared field. `SMTP` looks for a namespace `Conga` in the namespace that contains the package and in `#`, and copies it from the `conga` workspace if it cannot find it. To use a particular copy of Conga, set `CongaRef` to the Conga namespace (a reference or its name), or to an initialised Conga instance, before the first connection is made.

#### Config

Returns a matrix with the names and values of the instance's fields and properties.

### The "Attachment" class

Usually created for you by `MailParameters.AddAttachment`.

Create an instance with `⎕NEW Attachment(,⊂filename)` or `⎕NEW Attachment(filename mimeType)`. The file must exist. Without a MIME type it is worked out from the extension; an unknown extension results in `application/octet-stream`.

#### filename

Read-only property: the name of the file.

#### mimeType

The MIME type; can be changed.

#### GetMimeType, GetAllMimeTypes

Shared methods: `GetMimeType` returns the MIME type for an extension, `GetAllMimeTypes` a matrix with all known extensions and their MIME types.

### Other functions

#### SendEmailExample

Sends a test email; see "Send a test email" above. The left argument is an instance of `ConnectionParameters`, the right argument an email address, optionally followed by filenames.

#### Version

Returns the version number of the package as a text vector, or an empty vector when the package was not loaded by Tatin.

#### History

Returns the version history.

#### Help

Opens this document.
