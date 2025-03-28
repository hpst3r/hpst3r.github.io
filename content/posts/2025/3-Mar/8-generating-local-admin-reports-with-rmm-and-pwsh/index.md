---
title: "Adventures with secedit.exe and PowerShell, pt. 1"
date: 2025-03-27T12:13:59-00:00
draft: false
---

## I have a problem

Secedit is a bit confusing to use at first, so let's build a PowerShell wrapper to do our bidding!

## First, how do we use this darn thing?

After a brief detour [to MS Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/secedit) and some poking around...

Secedit is funky! It wants you to treat it with care and take it on long walks. No slamming random SIDs at it and making it figure the hard stuff out (at least, not yet)!

To use it to make changes to local security policy, you must:
1. Export a configuration database to a config file (`/export`)
2. Make your desired changes to the config file
3. Use the changes to create a configuration database (`/import`)
4. Configure the system with your changes (`/configure`)

Here's a demonstration! Let's set the `SeDenyNetworkLogonRight` privilege for a SID in the local Administrators group to prevent remote access by privileged users.

First, export the system's configuration to a file. We'll export just the USER_RIGHTS section of the database, since that's all we need.

```txt
Administrator in ~
❯ & secedit /export /cfg .\policy.cfg /areas USER_RIGHTS
```

Here's what that file looks like.

```txt
Administrator in ~
❯ gc sexport.cfg
[Unicode]
Unicode=yes
[Privilege Rights]
SeNetworkLogonRight = *S-1-1-0,*S-1-5-32-544,*S-1-5-32-545,*S-1-5-32-551
SeBackupPrivilege = *S-1-5-32-544,*S-1-5-32-551
SeChangeNotifyPrivilege = *S-1-1-0,*S-1-5-19,*S-1-5-20,*S-1-5-32-544,*S-1-5-32-545,*S-1-5-32-551
SeSystemtimePrivilege = *S-1-5-19,*S-1-5-32-544
SeCreatePagefilePrivilege = *S-1-5-32-544
SeDebugPrivilege = *S-1-5-32-544
SeRemoteShutdownPrivilege = *S-1-5-32-544
SeAuditPrivilege = *S-1-5-19,*S-1-5-20
SeIncreaseQuotaPrivilege = *S-1-5-19,*S-1-5-20,*S-1-5-32-544
SeIncreaseBasePriorityPrivilege = *S-1-5-32-544,*S-1-5-90-0
SeLoadDriverPrivilege = *S-1-5-32-544
SeBatchLogonRight = *S-1-5-32-544,*S-1-5-32-551,*S-1-5-32-559
SeServiceLogonRight = *S-1-5-80-0,*S-1-5-83-0,*S-1-5-99-0
SeInteractiveLogonRight = Guest,*S-1-5-32-544,*S-1-5-32-545,*S-1-5-32-551
SeSecurityPrivilege = *S-1-5-32-544
SeSystemEnvironmentPrivilege = *S-1-5-32-544
SeProfileSingleProcessPrivilege = *S-1-5-32-544
SeSystemProfilePrivilege = *S-1-5-32-544,*S-1-5-80-3139157870-2983391045-3678747466-658725712-1809340420
SeAssignPrimaryTokenPrivilege = *S-1-5-19,*S-1-5-20
SeRestorePrivilege = *S-1-5-32-544,*S-1-5-32-551
SeShutdownPrivilege = *S-1-5-32-544,*S-1-5-32-545,*S-1-5-32-551
SeTakeOwnershipPrivilege = *S-1-5-32-544
SeDenyInteractiveLogonRight = Guest
SeUndockPrivilege = *S-1-5-32-544,*S-1-5-32-545
SeManageVolumePrivilege = *S-1-5-32-544
SeRemoteInteractiveLogonRight = *S-1-5-32-544,*S-1-5-32-555
SeImpersonatePrivilege = *S-1-5-19,*S-1-5-20,*S-1-5-32-544,*S-1-5-6,*S-1-5-99-216390572-1995538116-3857911515-2404958512-2623887229
SeCreateGlobalPrivilege = *S-1-5-19,*S-1-5-20,*S-1-5-32-544,*S-1-5-6
SeIncreaseWorkingSetPrivilege = *S-1-5-32-545
SeTimeZonePrivilege = *S-1-5-19,*S-1-5-32-544,*S-1-5-32-545
SeCreateSymbolicLinkPrivilege = *S-1-5-32-544,*S-1-5-83-0
SeDelegateSessionUserImpersonatePrivilege = *S-1-5-32-544
[Version]
signature="$CHICAGO$"
Revision=1
```

Make your changes here! For example, to deny our user, I'll be adding the following line to the bottom of the Privilege Rights section:

```txt
SeDenyNetworkLogonRight = *S-1-5-21-2315843530-1563403064-648863213-1002
```

To load your modified configuration file into a database for use, use `secedit /import`:

```txt
Administrator in ~
❯ & secedit /import /db import.db /cfg policy.cfg
```

You can then configure the system with the database you just created with `secedit /configure`:

```txt
Administrator in ~
❯ & secedit /configure /db import.db

The task has completed successfully.
See log %windir%\security\logs\scesrv.log for detail info.

```

That log file looks something like this:

```txt
Administrator in ~
❯ gc $env:WinDir\security\logs\scesrv.log
-------------------------------------------
Thursday, March 27, 2025 8:33:44 PM
----Configuration engine was initialized successfully.----

----Reading Configuration Template info...


----Configure User Rights...
                SeImpersonatePrivilege must be assigned to administrators. This setting is adjusted.
                SeImpersonatePrivilege must be assigned to SERVICE. This setting is adjusted.
        Configure S-1-5-19.
        Configure S-1-5-20.
        Configure S-1-5-32-544.
        Configure S-1-5-32-551.
        Configure S-1-5-32-559.
        Configure S-1-1-0.
        Configure S-1-5-32-545.
        Configure S-1-5-6.
        Configure S-1-5-83-0.
        Configure S-1-5-21-2315843530-1563403064-648863213-501.
        Configure S-1-5-21-2315843530-1563403064-648863213-1002.
                add SeDenyNetworkLogonRight.
        Configure S-1-5-99-216390572-1995538116-3857911515-2404958512-2623887229.
        Configure S-1-5-90-0.
        Configure S-1-5-32-555.
        Configure S-1-5-80-0.
        Configure S-1-5-99-0.
        Configure S-1-5-80-3139157870-2983391045-3678747466-658725712-1809340420.

        User Rights configuration was completed successfully.


----Configure Group Membership...

        Group Membership configuration was completed successfully.


----Configure 64-bit Registry Keys...

        Configuration of Registry Keys was completed successfully.


----Configure 32-bit Registry Keys...


----Configure File Security...

        File Security configuration was completed successfully.


----Configure General Service Settings...

        General Service configuration was completed successfully.


----Configure available attachment engines...

        Configuration of attachment engines was completed successfully.


----Configure Security Policy...
        Configure password information.

        System Access configuration was completed successfully.

        Configuration of Registry Values was completed successfully.

        Audit/Log configuration was completed successfully.


----Configure available attachment engines...

        Configuration of attachment engines was completed successfully.


----Un-initialize configuration engine...

```

If we check the local security policy (via the secpol MMC snap-in), we can see our changes:

{{< figure src="images/secpolmmc.png" alt="Security Policy MMC showing SeDenyNetworkLogonRight on a system" >}}

Psst. We can skip the export step and manually write a config file, if needed:

```txt
Administrator in ~
❯ gc minimal.cfg
[Unicode]
Unicode=yes
[Privilege Rights]
SeDenyNetworkLogonRight = *S-1-5-21-2315843530-1563403064-648863213-1002
[Version]
signature="$CHICAGO$"
Revision=1

Administrator in ~
❯ & secedit /import /db import.db /cfg minimal.cfg

Administrator in ~
❯ & secedit /configure /db import.db

The task has completed successfully.
See log %windir%\security\logs\scesrv.log for detail info.
```

Anyway, that's all well and good, but I don't want to type stuff in manually! I'm much too lazy for that. Let's spend.. `New-Timespan -Hours (Get-Random -Maximum 744)` hours figuring out how to do it with PowerShell instead!

## Now, how do we make the computer use this darn thing?

Let's write a little function to add a privilege right! I should do one to remove privilege rights, but I'm too lazy to do this right now and I don't need it.

This is a bit more involved than it might first appear to be. We'll only support local users for now, to reduce some of the testing overhead, and I'm not going to do a ton of error handling as, frankly, it's past my bedtime.

Anyway.. here goes!

We'll drop ourselves into a UUID-labeled temporary directory...

```txt
$WorkingDir = (
	Join-Path `
		-Path $env:TEMP `
		-ChildPath (New-Guid)
	)

if (Test-Path $WorkingDir) {
	
	Remove-Item `
		-Path $WorkingDir `
		-Force `
		-Recurse

}

New-Item `
	-ItemType Directory `
	-Path $WorkingDir
```

Create a path for our config export, then call `secedit /export` with it:

```txt
$SecEditExportFile = (Join-Path -Path $WorkingDir -ChildPath 'Export.cfg')

# export existing database
& secedit.exe /export /cfg $SecEditExportFile
```

Now we can parse the export for the specific right we would like to make changes to with `Select-String` (regex).

The regular expression I'm using, `(?<=^$($Right) = ).*$`, can be broken down into three main bits:

- "`(?<=)`" is a positive lookbehind that will match text only if it follows the text after our `?<=` in parentheses.s
- "`^$($Right) = `" will match only if the beginning of the line is immediately followed by the PowerShell variable $Right, then the characters "` = `".
- "`.*$`" will match any number of any character until the end of the line.

This combination will match anything following "`Right = `", which happens to be our SIDs.

We could get a lot more particular about what to select, but there shouldn't be anything but SIDs here anyway.

```txt
$ExistingUsers = (
	Select-String `
		-Path $SecEditExportFile `
		-Pattern "(?<=^$($Right) = ).*$"
	).Matches.Value
```

Once we know what our existing SIDs (or usernames - Windows likes to resolve SIDs when it can!) are, we can see if our user is already there. If they are, we might as well exit (we'll clean up the temp directory in a `finally` block later).

```txt
if ($ExistingUsers -and (($User.SID.Value -in $ExistingUsers) -or ($User.Name -in $ExistingUsers))) {

	Write-Debug -Message `
		"Add-UserRightAssigment: User $($User.SID.Value) already has desired rights. No changes will be made."
	
	return
	
}
```

SIDs are sometimes (if resolution is possible) translated to names on import, so we have to translate them back if we want to preserve their rights assignments (and keep secedit from bombing out half the time). Unfortunately, we do want to make this work, so we have to do this.

This block splits the string of usernames/SIDs we selected two codeblocks ago on commas (see format of the export.cfg file - they're `,*` delimited SIDs, or comma delimited usernames) and cleans up extra asterisks, then gets the relevant SID by hitting `Get-LocalUser`. As `Get-LocalUser` has no trouble resolving SIDs, this works like a charm.

This needs some refactoring to handle AD groups, at a bare minimum - highly privileged groups are commonly added to this `SeDenyNetworkLogonRight` privilege.

Anyway, it then puts the list of SIDs back together, in a format that secedit likes, adding a trailing comma so we can plop our own SID at the end.

```txt
if ($ExistingUsers) {

	$SIDsFromExistingUsers = (
		$ExistingUsers -replace '\*','' -split ',' |
		Get-LocalUser |
		ForEach-Object {'*' + $_.SID.Value} |
		Join-String -Separator ','
	) + ',' # trailing comma for formatting with our SID to be added

}
```

Okay! Finally, we're getting somewhere! Let's generate a minimal secedit config file to import containing just what we want, so as to be good stewards of whatever this box is.

```txt
$NewConfigContent = @"
[Unicode]
Unicode=yes
[Privilege Rights]
$($Right) = $($SIDsFromExistingUsers)*$($User.SID.Value)
[Version]
signature="`$CHICAGO`$"
Revision=1
"@

# create a config file
$NewConfig = (
	New-Item `
		-ItemType File `
		-Path (
			Join-Path `
				-Path $WorkingDir `
				-ChildPath 'NewImport.cfg'
			)
	)

# set content of config file to minimal gen'd config (mline string above)
Set-Content `
	-Path $NewConfig `
	-Value $NewConfigContent

```

Before you know it, we're just about done! Let's create a database from our minimal config file, then apply it to the system.

```txt
$NewDB = (
	Join-Path `
		-Path $WorkingDir `
		-ChildPath 'NewImport.db'
	)

# create a temporary database with modified configuration
& secedit /import /db $NewDB /cfg $NewConfig

# apply the modified config db to the system config
& secedit /configure /db $NewDB
```

Sweet! That should be just about all we need for a basic implementation. Let's wrap it in a function:

```txt
Function Add-UserRightAssignment {
    param(
        [string]$Right,
        [Microsoft.PowerShell.Commands.LocalPrincipal]$User
    )

    try {

        $WorkingDir = (
            Join-Path `
                -Path $env:TEMP `
                -ChildPath (New-Guid)
            )

        if (Test-Path $WorkingDir) {
            
            Remove-Item `
                -Path $WorkingDir `
                -Force `
                -Recurse

        }

        New-Item `
            -ItemType Directory `
            -Path $WorkingDir

        $SecEditExportFile = (Join-Path -Path $WorkingDir -ChildPath 'Export.cfg')

        # export existing database
        & secedit.exe /export /cfg $SecEditExportFile

        # get line to modify
        $ExistingUsers = (
            Select-String `
                -Path $SecEditExportFile `
                -Pattern "(?<=^$($Right) = ).*"
            ).Matches.Value

        # don't make any changes if user already exists
        if ($ExistingUsers -and (($User.SID.Value -in $ExistingUsers) -or ($User.Name -in $ExistingUsers))) {

            Write-Debug -Message `
                "Add-UserRightAssigment: User $($User.SID.Value) already has desired rights. No changes will be made."
            
            return
            
        }

        # if there's stuff where SIDs are, look up local users
        # and convert names to SIDs - or schtuff no worky
        if ($ExistingUsers) {

            $SIDsFromExistingUsers = (
                $ExistingUsers -split ',' |
                Get-LocalUser |
                ForEach-Object {'*' + $_.SID.Value} |
                Join-String -Separator ','
            ) + ',' # trailing comma for formatting with our SID to be added

        }

        # generate a new minimal config with desired SID
        # extra space on line 4 is OK
        $NewConfigContent = @"
[Unicode]
Unicode=yes
[Privilege Rights]
$($Right) = $($SIDsFromExistingUsers)*$($User.SID.Value)
[Version]
signature="`$CHICAGO`$"
Revision=1
"@
    
        # create a config file
        $NewConfig = (
            New-Item `
                -ItemType File `
                -Path (
                    Join-Path `
                        -Path $WorkingDir `
                        -ChildPath 'NewImport.cfg'
                    )
            )

        # set content of config file to minimal gen'd config (mline string above)
        Set-Content `
            -Path $NewConfig `
            -Value $NewConfigContent

        $NewDB = (
            Join-Path `
                -Path $WorkingDir `
                -ChildPath 'NewImport.db'
            )

        # create a temporary database with modified configuration
        & secedit /import /db $NewDB /cfg $NewConfig

        # apply the modified config db to the system config
        & secedit /configure /db $NewDB

    }
    finally {

        # clean up working directory
        Remove-Item `
            -Path $WorkingDir `
            -Force `
            -Recurse

    }

}
```

Now, let's use it!

```txt
Add-UserRightAssignment -Right 'SeDenyNetworkLogonRight' -User (Get-LocalUser liam)
```

Wasn't that easy?? Works like a charm! Until you look at it wrong.

That's all for now. TBC...