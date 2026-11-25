# SMTP

`SMTP` allows you to send emails from within Dyalog APL: plain text or HTML, with or without attachments, through an SMTP server of your choice.

**Note:** `SMTP` releases are published as [Tatin](https://tatin.dev "Link to the principal Tatin Registry") packages, see there.

You need an account with an SMTP server that offers SSL/TLS (usually on port 465) or an unencrypted connection. STARTTLS (usually port 587) is not supported yet. Earlier versions came with a shared Gmail account for trying things out; that account no longer exists.

1. Load the package into the workspace with

   ```
   ]TATIN.LoadPackage [tatin]SMTP
   ```

2. Send yourself a test email with your own credentials:

   ```
   cp←⎕NEW SMTP.ConnectionParameters
   cp.Server←'smtp.example.com'
   cp.Userid←'me@example.com'
   cp.Password←'my-app-password'
   cp SMTP.SendEmailExample 'john.doe@whatever.com'
   ```

With Gmail, for example, the server is `smtp.gmail.com`, and the password is an "app password" that you can create in the security settings of a Google account with 2-Step Verification switched on.

Do not put the password into a file that is under version control, and do not hard-code it into code you distribute: anybody who finds it can send email in your name.

`SMTP` is well suited for sending notifications from a server application, for example about

* Crashes
* "Start" and "Shutdown" events

Email is not end-to-end encrypted, so if nobody but the recipient may read what you send, encrypt the body, and any attachments, before sending them.

Further information is available by loading the package and then calling the `Help` method.
