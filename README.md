# **Webmail**

## Overview

### What is Roundcube? 

**Roundcube** is a free and open-source software web-based IMAP email client with an application-like user interface. It provides full functionality expected from an email client, including MIME support, address book, folder manipulation, message searching and spell checking.

## Administrator Guide

### Virtual Domain 

- Postfix contains a list of virtual domains that will accept mail for and deliver them to Dovecot.
- Dovecot will accept any and all destination addfresses and domains.
- Settings for this configuration are in the users section of the `dovecot.conf` file as well as the `Lua Script` file, which could be modified to enforce a list of valid usernames.

### Contacts

To import and export contacts, refer to this link:
- [Roundcube Import/Export Contacts] (https://docs.roundcube.net/doc/help/1.1/en_US/addressbook/importexport.html)

- Addresses added to contacts are inserted into the contacts table in the VCARD format with a field specified for the `user_id` that added the contact to their address book.
- User identities are added to the identities table after first logging in.
- Addresses are added to the `collected_addresses` table after an email is sent to the address.
- Default groups are personal, collected, and trusted.
- Contact groups can only be added under the default personal group and will be added to the contact groups table with indication of the `user_id` that owns the group.

### Global Addressbook

To configure the addressbook, refer to these two links:

- [Roundcubemail Example Addressbook] (https://github.com/roundcube/roundcubemail/blob/master/plugins/example_addressbook/example_addressbook.php)
- [Roundcube Webmail GlobalAddressbook] (https://github.com/johndoh/roundcube-globaladdressbook)

1. Use `git` to download the code.
2. Copy the sample config file and make any desired edits of value.
3. Use the admin accounts to import a CSV of account names into the addressbook.

### Notifications

To add the notifications feature, refer to this link:

- [Roundcube HTML5 Notifier] (https://packagist.org/packages/kitist/html5_notifier)

### Mail Reply Options

The mail reply options are:

- `htmleditor = 1` - setting to enable HTML editor.
- `replymode = 1` - setting to reply on top of email instead of at the bottom of the email.

### Real & Fake Email Configuration

Follow these steps to configure fake emails from real user emails:

1. Enter the Roundcube pod, navigate to `var/www/html/config` and open the `config.inc.php` file.
2. Set the value of `oauth_identity_fields` to `username`.
3. Then, set `username_domain` to your desired email domain.
4. Set `force_username_domain` to `true`.
5. On the Dovecot pod, navigate to `etc/dovecot` to update the dovecot oauth settings.
6. Open the `dovecot.conf` file and set `username_format = %n`, which will allow it to just look at the username, so a mismatch in the username (email) given by roundcube doesn't need to match the username (email) given by the token introspection URL.

### Email Autoreply

Determine whether an address is a simulated user that comes from the roundcube global address book database.

1. Navigate to `/etc/postfix/autoreply`
2. Add the following code:

`#!/bin/bash

sender=$1
recipient=$2

org = `echo $recipient | cut -d "@" -f 1`

/usr/sbin/sendmail - F $org -f $recipient $sender < /etc/postfix/response`

3. Add the following to `/etc/postfix/response`:

'Add your desired response for autoreplies from simulated users`

### Adding EXERCISE Banner to Email

1. Access postfix's Kubernetes pod.
`kex postfix`
2. Go to this directory:
`cd /etc/postfix`
3. Here, we need to edit the `main.cf` file. To edit it:
`vim main.cf`
4. Now, we need to look for: `header_checks` variable, uncomment it and assign it the following value. (This will help us add "EXERCISE" to the subject field of the email).
`header_checks=pcre:/etc/postfix/maps/rewrite_headers`
5. After, click `ESC` and save it.
6. Now, we need to configure some fields in the `master.cf` file to add "EXERCISE" to the header and footer of the email.
`vim master.cf`
7. Under the table of service types add the following line. (This will help us add "EXERCISE" to the email body's header and footer).
` filter    unix  -       n       n       -       100     pipe
    flags=Rq user=filter null_sender=
    argv=/etc/postfix/maps/filter -f ${sender} -- ${recipient}
-o content_filter=filter:dummy`
8. After, click `ESC` and save it.
9. Navigate to the maps folder.
`cd /etc/postfix/maps`
10. Create a file called `rewrite_headers`, with the following contents to add the word EXERCISE to the subject of the email:
`if !/^Subject:.*\[EXERCISE EXERCISE EXERCISE\].*$/
/^(Subject: )(.+)$/i REPLACE $1 [EXERCISE EXERCISE EXERCISE] $2
endif`
11. After, click `ESC` and save it.
12. Then, create another file called `filter`, with the following contents to add the word EXERCISE to the email body's header and footer:

```
#!/bin/bash
echo 'date' >> /tmp/test
\# Simple shell-based filter. It is meant to be invoked as follows:
\#       /path/to/script -f sender recipients...
\# Localize these. The -G option does nothing before Postfix 2.3.
INSPECT_DIR=/var/spool/filter
SENDMAIL="/usr/sbin/sendmail -G -i" \# NEVER NEVER NEVER use "-t" here.
\# Exit codes from <sysexits.h>
EX_TEMPFAIL=75
EX_UNAVAILABLE=69
\# Clean up when done or when aborting.
trap "rm -f in.$$" 0 1 2 3 15
\# Start processing.
cd $INSPECT_DIR || {
echo $INSPECT_DIR does not exist; exit $EX_TEMPFAIL; }
cat >in.$$ || {
echo Cannot save mail to file; exit $EX_TEMPFAIL; }
\# plain text message from roundcube
perl -0pe 's/(Content-Type: text\/plain.*flowed)(.*)(--=_.*Content-Transfer-Encoding:)/\1\n\nEXERCISE EXERCISE EXERCISE\2\n\nEXERCISE EXERCISE EXERCISE\n\3/s' in.$$ > in.$$-2
mv in.$$-2 in.$$
\# html message from roundcube
perl -0pe 's/(Content-Type: text\/html.*<body.*serif.>)(.*)(.*<\/body>)/\1\nEXERCISE EXERCISE EXERCISE\n\n\2\n\nEXERCISE EXERCISE EXERCISE\3/s' in.$$ > in.$$-2
mv in.$$-2 in.$$
\# base64 message from stackstorm 
perl -0pe 's/(.*Content-Transfer-Encoding: base64\n\n)(.*)(\n\n--===============.*)/\2/ms' in.$$ > in.$$-base64
cat in.$$-base64 | base64 -d > in.$$-msg
echo -n "EXERCISE EXERCISE EXERCISE" > in.$$-newmsg
cat in.$$-base64 | base64 -d >> in.$$-newmsg
echo "EXERCISE EXERCISE EXERCISE" >> in.$$-newmsg
cat in.$$-newmsg| base64 > in.$$-new64
NEW64='cat in.$$-new64'
perl -0pe "s/(.*Content-Transfer-Encoding: base64\n\n)(.*)(\n\n--===============.*)/\1\n$NEW64\n\n\3/ms" in.$$ > in.$$-2
mv in.$$-2 in.$$
\# finished processing
$SENDMAIL "$@" <in.$$
exit $?
```

13. After, click `ESC` and save it.
14. After all configurations have been made, execute the following command to apply all changes:
`postfix reload`

### Rejecting Invalid Email Addresses

1. Access Postfix's Kubernetes pod:
`kex postfix`
2. Go to this directory:
`cd /etc/postfix`
3. Here, we need to edit the `main.cf` file. To edit it:
`vim main.cf`
4. Add this line to the file:
`virtual_mailbox_maps = pgsql:/etc/postfix/pgsql-virtual-recipient-maps.cf`
5. After, click `ESC` and save it.
6. Navigate to the maps folder.
`cd /etc/postfix/maps`
7. Then, create a file called `pgsql-virtual-recipient-maps.cf` and add the following contents to the file:
`dbname = //add the name of the database
table = //add the name of the table
select_field = email
where_field = email
hosts = //add the hosts connection to the database
query = SELECT email FROM contacts WHERE email='%s'`
8. After, click `ESC` and save it.
9. After all configurations have been made, execute the following command to apply all changes.
`postfix reload`

## Plugins

### Message History Plugin

Upon activation, this plugin will generate an exclusive administrative area that can only be accessed by users holding the Observer role. Within this section, a table will be presented, which will assemble data concerning sent and received emails, including pertinent information such as sender, recipient, subject, date, time, read status, and message identification number. Emails sent will be recorded in the table and once the intended recipient reads the message, the table will reflect this by marking the email as "read".

#### Plugin Activation

To enable this plugin, follow these steps:

1. Enter the Roundcube pod.
2. Navigate to `/var/www/html/config`.
3. Open the `config.inc.php` file.
4. Search for the plugins sections, and add `message_history` to the plugin list.

#### Plugin Access Group

To change the group of users who can view this administrative area, follow these steps:

1. Enter the Roundcube pod.
2. Navigate to `/var/www/html/plugins/message_history`.
3. Open the `config.inc.php` file.
4. Change the name of the group created on the global address book.
5. After, click `ESC` and save it.

### xAPI Plugin


