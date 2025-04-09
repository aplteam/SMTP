# SMTP

`SMTP` allows you to send emails from within Dyalog APL. It comes with a ready-to-go example that works out of the box.

**Note:** `SMTP` releases are published as [Tatin](https://tatin.dev "Link to the principal Tatin Registry") packages, see there.


1. To try it out, load the package into the workspace with 

   ```
   ]TATIN.LoadPackage [tatin]SMTP
   ```

2. Send an email with 

   ```
   SMTP.SendEmailExample 'john.doe@whatever.com'
   ```

Note that the example uses a particular email address and an API key that anybody who loads the package can access. Of course this means that you must not send anything remotely confidential to this email address as clear text.

This address is configured so that an application can kind of log on with that API key, and then send emails to that address; this is considered relatively unsafe, and it is not recommended by Google. This is also the reason why an email send via this sender address is likely to be qualified as spam by Google.

It cannot be used for spam because the number of emails one can send this way is severely limited by Google, but it can be used to send a couple of emails.

Nobody is watching that email address, but the _real_ receiver is the CC anyway of course.

You may create your own email address (and API key) with Google and use it to send messages from a server for reporting stuff like

* Crashes
* "Start" and "Shutdown" events

If you need to make sure that nobody can decipher any mail you are going to send this way then you must first encrypt the body and possibly any attachments as well.

Further information is available by loading the package and then calling the `Help` method.

