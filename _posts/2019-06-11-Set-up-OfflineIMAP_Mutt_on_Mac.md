---
layout: post
title: Set up OfflineIMAP + Mutt on Mac
tags: [Tool, Mac]
---

# Summary
Recently, I submitted my first kernel patch. One of the lessons I learned is
that kernel mail server doesn’t accept HTML content in emails. To make sure my
email contains absolutely no HTML, I set up a text-based email client (Mutt +
OfflineIMAP) on my laptop. This post is about my configs that make the email
client as close to a desktop client (like Outlook or Mail app on Mac) as
possible.

# Basic Configs
[Mutt](http://www.mutt.org/) is a powerful text-based email client.
[OfflineIMAP](http://www.offlineimap.org/) is a Python utility to sync mail
from IMAP servers. In my setup, I use OfflineIMAP to periodically sync a local
mail folder with remote outlook email server and use Mutt as an email client to
read emails from the folder.

![OfflineIMAP and Mutt](/img/offlineimap_mutt.jpg){: .center-block :}

## OfflineIMAP: Install and Config
### Install
```
$ brew install offline-imap
```


### Config
After installation, a minimal `.offlineimaprc` will be created for you. You
only have to adapt it. For example:
```
[general]
accounts = Personal
pythonfile = ~/.offlineimap.py

[Account Personal]
localrepository = Local
remoterepository = Remote

[Repository Local]
type = Maildir
localfolders = ~/Maildir/Personal

[Repository Remote]
type = IMAP
remotehost = outlook.office365.com
remoteuser = <my-email-address>
remotepasseval = get_pass()
ssl=true
sslcacertfile = /usr/local/etc/openssl/cert.pem
```

### Get Password from Keychain
To avoid putting my password in plain text, I wrote a python function to get
password from keychain. The function is defined in `~/.offlineimap.py`. This
path must be specified in .offlineimaprc by adding `pythonfile =
~/.offlineimap.py` in `[general]` section.
```
$ cat ~/.offlineimap.py
#! /usr/bin/env python3
from subprocess import check_output

def get_pass():
    return check_output("security find-internet-password -w -a <my-email>", shell=True).strip("\n")
```

This password was added to keychain by command:
```
$ sudo /usr/bin/security -v add-internet-password -a <my-email> -s outlook.office365.com
```

### Run it periodically
After the config is done, run `offlineimap` command, all mails from remote IMAP
server will be sync’d to `~/Maildir/Personal`. But we are not done yet because
`offlineimap` has to run periodically in order to keep in sync with the mail
server. To do so, we can leverage `launchd`. I didn’t choose `crontab` because
`crontab` man page on Mac says:

> (Darwin note: Although cron(8) and crontab(5) are officially supported under
> Darwin, their functionality has been absorbed into launchd(8), which provides
> a more flexible way of automatically executing commands. See launchctl(1) for
> more information.) 

A good thing about using `Homebrew` to install `OfflineIMAP` is, it creates a
service for you so that you don’t have to write one yourself. To start the
service, simply run:
```
$ brew services start offlineimap
```
The default interval is 300 seconds (5 minutes), which is too long compared to
other desktop email clients. So I changed it to 20 seconds by modifying
`StartInterval` in
`/usr/local/Cellar/offlineimap/7.2.3/homebrew.mxcl.offlineimap.plist`.

Till this point, we have finished setting up OfflineIMAP with most basic
configs, which are enough for a noob like me. For more details, see
[offlineIMAP documentation](http://www.offlineimap.org/documentation.html).

## Matt: Install and Config

### Install
```
$ brew install mutt
```

### Config
All mutt configs are in `~/.muttrc` file. For an email client, two most basic
requirements are sending and receiving emails. For outgoing emails, I use SMTP.
For incoming emails, I set mailbox type to Maildir so that Mutt will read
emails from the specified folder to which OfflineIMAP writes.

```
# SMTP
set smtp_url = "smtp://<my-email>@smtp.office365.com:587"
set smtp_pass = "`security find-internet-password -w -a <my-email>`"
set realname = "Hechao Li"
set from = "<my-email>"
set attribution = {% raw %}"%n <%a> wrote on %{%a} [%{%Y-%b-%d %H:%M:%S %Z}]:"{% endraw %}

# IMAP
set mbox_type = Maildir
set folder = "~/Maildir/Personal"
set mbox = "+INBOX"
set postponed = "+Drafts"
set spoolfile = "+INBOX"
set record = "+Sent Items"
set trash = "+Deleted Items"
set timeout = 5   # Sync every 5 seconds
```

For details of these variables, see [Mutt
manual](https://muttmua.gitlab.io/mutt/manual-dev.html).

Till this point, we have finished setting up Mutt as a basic email client and
we are able to send and receive emails. 

# Advanced Configs
As a person who is spoiled by GUI, I still want to use Mutt like using a
desktop email client. So I spent some time figuring out how to set up the
following features.
## Mailboxes
I want my mailboxes shown on the left like all other GUI email clients. Because
I don’t want to hardcode all mailboxes in `.muttrc`, I will ask OfflineIMAP to
populate them for me. Add the following config to `.offlineimaprc`:

```
[mbnames]
enabled = yes
filename = ~/.mutt/mailboxes
header = "mailboxes "
peritem = "+%(foldername)s"
sep = " "
footer = "\n"
```

And add the following configs to `.muttrc`:
```
source ~/.mutt/mailboxes
set sidebar_visible = yes
set sidebar_format = "%B%*  %N" # Show number of unread messages
```

## View HTML Email
Though mutt guarantees that I won’t send any HTML content, I can’t prevent
people/systems from sending HTML emails to me. Therefore I need a way to view
them. I use two tools: [w3m](http://w3m.sourceforge.net/), a text based web
browser and [muttils](https://github.com/cwarden/muttils), a set of utilities
for Mutt. The former can help me browse simple HTML in plain text in Mutt. And
for fancier HTML emails that don’t look good in a text-based browser, I will
use the latter to view them in a real web browser like Chrome.

### w3m
According to [Mutt
manual](https://muttmua.gitlab.io/mutt/manual-dev.html#mailcap),

> In order to handle various MIME types that Mutt doesn't have built-in support
> for, it parses a series of external configuration files to find an external
> handler.

w3m is installed by `brew install w3m`. And the configuration for MIME types
are written in a "mailcap" file.

In `~/.mutt/mailcap`:
```
set text/html; w3m -I %{charset} -T text/html; copiousoutput;
```

In `.muttrc`:
```
set mailcap_path  = ~/.mutt/mailcap
auto_view text/html
alternative_order text/plain text/enriched text/html
```

See [this](https://muttmua.gitlab.io/mutt/manual-dev.html#auto-view) for
details of above config. 

### Muttils
Similarly, muttils is installed by `brew install muttils`. After installation,
add a macro to `.muttrc`.

```
$ macro pager G "<pipe-message>viewhtmlmsg<enter>"
```
Then when a fancy HTML email is received, I can simply press "G" to view it in
Google Chrome. See [here](https://muttmua.gitlab.io/mutt/manual-dev.html#macro)
for more information about macros in Mutt.

## View Attachments
Similar to the way to view HTML emails, we need to add a config in
`~/.mutt/mailcap`. For example, to open PDF attachments in *Preview*:
```
application/pdf; open -a Preview '%s'; copiousoutput;
```

For now I only have a config for PDF attachments but extending it to support
more document types shouldn’t be too hard.

## Notification
I want to receive a notification when a new email arrives.
[Terminal-notifier](https://github.com/julienXX/terminal-notifier) is what I
use. Again, install it via Homebrew: `brew install terminal-notifier`. Then add
the following config to `.muttrc`:

```
set new_mail_command="terminal-notifier -title '%v' -subtitle 'New Mail' -message '%n new messages, %u unread.'"
```

## Make it "Vimmy"
Since I am so used to vim, instead of memorizing new operations in Mutt, I can
use macros to make key mappings more "vimmy". [This
article](https://ryanlue.com/posts/2017-05-21-mutt-the-vim-way) has some useful
bindings. In short, make friends with macros and bindings so that you can
customize Mutt.

## Calendar (TBD)
I want to view and respond to event invite in Mutt but I haven't figured out a
good way yet. For now, I will just use Apple calendar to accept/reject invites.
