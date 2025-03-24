---
title: "Updating multiple M365 mailboxes' autoreply settings"
date: 2025-03-20T12:13:59-00:00
draft: false
---

Quick one for a personal reminder.

You can also schedule an autoreply by setting `State` to `Scheduled`, and providing a `StartTime` and `EndTime`.

```txt
# set autoreply for multiple users

$Message = @"
This mailbox is no longer monitored.<br>
<br>
If you require assistance, please contact so-and-so in the Doing Stuff office.<br>
<br>
Thank you,<br>
IT
"@

@('user1@domain.net','user2@domain.net') | % { 
    Set-MailboxAutoReplyConfiguration `
        -Identity $_ `
        -AutoReplyState Enabled `
        -ExternalMessage $Message `
        -InternalMessage $Message `
        -ExternalAudience All
}
```