[parm]:numberHeaders     = 2 3 4 5 6
[parm]:leanpubExtensions = 1



# `SMTP` Help!

## Overview

Sending emails from APL code (or any code) is, thanks to the spammers, surprisingly difficult.

However, there is a solution: the `SMTP` package from the Tatin server. It works out of the box; the README.md document carries information for how to send yourself an email with a single command after loading the package.

A> ### It might not work
A>
A> A few users have reported problems when trying to use `SendEmailExample` or running the test cases. It appears that Google has some AI in place that tries to figure out when somebody is using the credentials without actually being entitled to do so, and that results in stuff not working, and there is not much we can do.
A>
A> If this affects you then we suggest that you create your own Email account and use that; see "Using smtp.google.com yourself" below.

## User Guide

### Preconditions

You are free to use any public SMTP server of course, but in the function `SendEmailExample` as well as in its test cases `SMTP` uses Google's SMTP server `https://smtp.google.com`.

### Send an email out of the box

You can use the `SendEmailExample` function in order to send emails out of the box:

```
   SMTP.SendEmailExample 'john.doe@whatever.com'
```

That will send a simple test email to the address provided.

You may add one or more files as attachments:

```
   ('my.txt' 'foo.htm') SMTP.SendEmailExample 'john.doe@whatever.com'
```

This uses hard-coded credentials. The email address is only suitable for tests, don't use it for anything else!
Be aware that the credentials might well change in case somebody abuses this, or the email address might disappear altogether.


### Using smtp.google.com yourself

It's easy enough to take advantage of smtp.google.com youself. The first thing you need is a Gmail address. 

We suggest to create a new one because that way you don't need to worry too much about safety. The worst thing that can happen is that you lose that GMail address. Well, just create a new one then.

Of course this implies that you should not use such an email address for anything remotely confidential, but for sending, say, a crash notification it's absolutely fine.

However, just creating the email address is not enough: you also need to allow access in a way that Google considers unsafe, which is the reason why it is not active by default.

In order to enable it follow these steps:

1. Log on

2. Select "Setting" ==> "See all setting"

3. Go to the tab "Accounts and Import"

4. Under "Change account settings" click "Other Google Account settings"

5. On the left click "Security"

6. Make sure that 2FA (two-factor authentication) is active because this is a requirements for the next step to be available.

7. In the section "Signing in to Google" there is an option "App password" available

8. From the "Select app" drop-down select "mail"

9. From the "Select Device" drop-down select "Other (custom name)" and enter something meaningful, like "MyTatinServer"

10. Click "Generate"

A dialog box opens with a 16-character password. Make sure to copy this password into the INI file of your Tatin server in the `[EMAIL]` section and you are ready to go.


A> ### Less secure app access
A>
A> In the past it was possible to allow Apps access via the "Less secure Apps" option. This is not available anymore since March 2022. 
A>
A> The reason is that somebody had full access to the Google account when logging on via this mechanism.

**Note that this is a valid description as of 2021-09.** It might well change, and this recipe is not going to be updated. However, you should be able to find your way as long as the general machanism is still working.

A> ### Create your own GMail address for SMTP
A>
A> 1. When being logged on to your Gmail account, click on the avatar symbol at the top right corner.
A>
A>   A dialog box pops up. 
A>
A> 2. Select "Manage your Google account".
A>
A>   The "Google account" page appears. 
A>
A> 3. Click on "Security" on the left site.
A>
A>   The "Security" page appears,
A>
A> 4. Make sure the 2-Step Verification in the "Signing in to Google" section is ative; without this being active you cannot carry out the next step.
A>
A> 5. Select "App passwords", also in the "Signing in to Google" section.
A>
A> 6. You will be prompted for you password (again).
A>
A>    The "App password" page pops up.
A>
A> 7. Click on "Select app" and select "Other (custom name)" from the selection list.
A>
A> 8. Enter an app name. This name is not really used for anything, its purpose is tell you what this is all about. 
A>
A>    Don't forget that you will probably forget why you do right now what you do, so enter something that tells you enough to jog your memory in a distant future.
A>
A> 9. Enter a device name. Same consideration as for topic number 8 applies here.
A>
A> 10. Finally you get the "generated app password" page with the password --- job done.

### Usage

Steps to send an email:

1. Create an instance of the `ConnectionParameters` class.

2. Make amendments according to your needs.

3. Create an instance of the `Connection` class by providing the instance of the `ConnectionParms` class.

4. Create an instance of the `MailParameters` class.

5. Make amendments according to your needs.

6. Optionally add attachments. 

   This can be done by calling the `AddAttachment` method of the instance of the `MailParameters` class. In case no instance of the `AddAttachment` is passed as argument to `AddAttachment` it will internally create one from the argument.

   `AddAttachment` accepts three different arguments:

   * The attachment's filename. This is particularly useful under Windows because the `Attachment` class will work out the appropriate MIME type according to the extension.

   * The attachment's filename and its MIME type as two text vectors. Use this in case there is no extension.

   * An instance of the `Attachment` class.

7. Call the `Send` method of the instance of the `Connection` class and pass the instance of the `MailParameters` class as an argument. This method will send the email.

Notes:

* When properties of the classes `ConnectionParameters`, `MailParameters` and `Attachment` are set or changed they are checked straight away, so the user will be told immediately when she did something wrong.

* Both classes `ConnectionParameters` and `MailParameters` offer a method `PerformChecks` which makes sure that all required properties have reasonable values, and that the setting of the properties is consistent.

  You don't have to call these methods because `SMTP` will do this anyway, but you may if you wish.



## Reference

### The "ConnectionParameters" class

An instance of this class must be passed as argument to `⎕NEW Connection`.

#### Server

Server address (something like "smtp.google.com")

#### Password

Optional password (if server requires authentication) 

#### Userid

Used for authentication (defaults to `From`)

#### Secure

| 1 | SSL/TLS
| 0 | not secure

#### Port

Defaults to 465

#### Domain

Fully qualified domain name for logging on with `Userid` and `Password`

#### Org

Optional organization

#### ReplyTo

Optional reply to email address

#### XMailer

Carries the text vector "Dyalog SMTP Client {versionNumber}`

#### TLSFlags

Defaults to 32 which stand for: accept server certificate without validating.

See Conga User Guide Appendix C.


### The "MailParameters" class

An instance of this class must be passed as argument to the instance method `Send` of the `Connection` class.


#### To

Simple ASCII text vector representing an email address.


#### CC

A simple ASCII text vector representing one or more email addresses. Multiple addresses must be separated by commas.


#### BCC

A simple ASCII text vector representing one or more email addresses. Multiple addresses must be separated by commas.

#### From

A simple ASCII text vector representing an email address.

#### Subject

Simple UTF-8 text vector

#### Body

Either a simple UTF-8 text vector or a vector of simple UTF-8 text vectors,

#### Attachments

Empty or a vector of instances of the `Attachment` class, added by the `AddAttachment` method.