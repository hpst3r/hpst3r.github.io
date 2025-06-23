---
title: "Using the Dell Command | Update CLI to update drivers from PowerShell"
date: 2025-06-23T14:30:00-00:00
draft: false
---

Dell Command | Update is the only piece of Dell software that I intentionally put on machines. It's one of the two useful Dell apps, alongside the Power Manager applet. It's a driver manager that can be used to fetch the most recent drivers validated and published for a piece of Dell hardware, and it's a nice utility to have.

It's better (in my opinion) than HP and Lenovo's options (Support Assistant and Commercial Vantage, respectively) because it's available via the Winget packages `Dell.CommandUpdate` (Classic, v4.6) and `Dell.CommandUpdate.Universal` (.NET 8, v5.5), and it has a usable CLI interface that can be used to configure the software with PowerShell!

You can also use the Settings menu in DC|U itself to make the same tweaks on a smaller number of machines - just run it as Administrator, or everything will be greyed out.

The `dcu-cli` binary can be found under either C:\Program Files or C:\Program Files (x86) depending on the specific version of Dell Command installed. You can call it like any other binary executable from `cmd.exe` or PowerShell:

```sh
& 'C:\Program Files\Dell\CommandUpdate\dcu-cli.exe' /configure -scheduleWeekly=Mon,23:45
```

### Useful arguments

To configure Dell Command | Update to run on a schedule:

```sh
/configure -schedule(Daily|Weekly|Monthly)=Date,Time
e.g.:
/configure -scheduleMonthly=first,Sun,00:45
/configure -scheduleWeekly=Mon,00:45
/configure -scheduleDaily=00:45
```

To set the system to automatically update without notifying the user:

```sh
/configure -updatesNotification=disable -scheduleAction=DownloadInstallAndNotify
```

To kick off an immediate update:

```sh
/applyUpdates
/applyUpdates -reboot
```

Set the system to update once a month, on the first Wednesday, at 00:30 without prompting the user:

```sh
/configure -scheduleMonthly=first,Weds,00:30 -updatesNotification=disable -scheduleAction=DownloadInstallAndNotify
```

To reset Dell Command | Update settings to their defaults:

```sh
/configure -restoredefaults
```

### Run with PowerShell

If you're running this from RMM, it's probably best to call `dcu-cli` from Start-Process.

```PowerShell
Start-Process 'C:\Program Files\Dell\CommandUpdate\dcu-cli.exe' -ArgumentList '/configure -scheduleMonthly=first,Sun,00:45 -updatesNotification=disable -userConsent=disable' -NoNewWindow
```

#### Handling x86 and amd64 versions

As previously mentioned, some versions are installed to Program Files (x86), and others live under normal Program Files. To handle this discrepancy, you can use the Resolve-Path cmdlet:

```PowerShell
$DcuPath = (Resolve-Path "C:\Program Files*\Dell\CommandUpdate\dcu-cli.exe")
```

```PowerShell
PS C:\WINDOWS\system32> $DcuPath

Path
----
C:\Program Files (x86)\Dell\CommandUpdate\dcu-cli.exe
```

Using this with Start-Process is simple enough:

```PowerShell
$DcuPath = (Resolve-Path "C:\Program Files*\Dell\CommandUpdate\dcu-cli.exe")

Start-Process $DcuPath -ArgumentList '/configure -scheduleMonthly=first,Sun,00:45 -updatesNotification=disable -scheduleAction=DownloadInstallAndNotify' -NoNewWindow
```

E.g.:

```PowerShell
PS C:\WINDOWS\system32> $DcuPath = (Resolve-Path "C:\Program Files*\Dell\CommandUpdate\dcu-cli.exe")
PS C:\WINDOWS\system32> Start-Process $DcuPath -ArgumentList '/configure -scheduleMonthly=first,Sun,00:45 -updatesNotification=disable -scheduleAction=DownloadInstallAndNotify' -NoNewWindow
'-scheduleMonthly' setting updated with value 'first,Sun,00:45'.
'-updatesNotification' setting updated with value 'disable'.
'-scheduleAction' setting updated with value 'DownloadInstallAndNotify'.
Settings were modified at 6/23/2025 12:45:58 PM
Execution completed.
The program exited with return code: 0
```

#### Example output on a 7320 with DC|U Universal 5.5 (.NET 8)

Configure DC|U:

```PowerShell
PS C:\Program Files\Dell\CommandUpdate> Start-Process 'C:\Program Files\Dell\CommandUpdate\dcu-cli.exe' -ArgumentList '/configure -scheduleMonthly=first,Sun,00:45 -updatesNotification=disable -scheduleAction=DownloadInstallAndNotify' -NoNewWindow
'-scheduleMonthly' setting updated with value 'first,Sun,00:45'.
'-updatesNotification' setting updated with value 'disable'.
'-scheduleAction' setting updated with value 'DownloadInstallAndNotify'.
Settings were modified at 6/23/2025 12:42:21 PM
Execution completed.
The program exited with return code: 0
```

Reset DC|U settings:

```PowerShell
PS C:\Program Files\Dell\CommandUpdate> Start-Process 'C:\Program Files\Dell\CommandUpdate\dcu-cli.exe' -ArgumentList '/configure -restoreDefaults' -NoNewWindow
Restored settings to program defaults.
Settings were modified at 6/23/2025 11:25:47 AM
Execution completed.
The program exited with return code: 0
```

Apply updates:

```PowerShell
PS C:\Program Files\Dell\CommandUpdate> Start-Process 'C:\Program Files\Dell\CommandUpdate\dcu-cli.exe' -ArgumentList '/applyUpdates' -NoNewWindow
PS C:\Program Files\Dell\CommandUpdate>
Checking for updates...
Checking for application component updates...
Scanning system devices...
Determining available updates...
1 updates were selected. Download Size: 12.7 MB
[1] TXNTH, Intel Wireless Authentication and Privacy Infrastructure Driver, 22.2150.0.1
Scanning system devices...
Downloaded updates (1 of 1)., 12.7 MB of 12.7 MB transferred (100.00%)...
Downloaded updates (0 of 0)., 12.7 MB of 12.7 MB transferred (100.00%)...
Installing updates (1 of 1). Update Name: Intel Wireless Authentication and Privacy Infrastructure Driver
Finished installing the updates.
1 of 1 update(s) successfully installed.
The system has been updated.
Execution completed.
The program exited with return code: 0
```

### Full help page

```txt
PS C:\WINDOWS\system32> & 'C:\Program Files\Dell\CommandUpdate\dcu-cli.exe' /help

Dell Command | Update v5.4.0
Usage:

dcu-cli.exe /<command> [-<option1>=<value1>] [-<option2>=<value2>]...

Commands:
    /help - Displays usage information.
    /? - Displays usage information.

    /scan - Performs a system scan to determine the updates for the current system configuration.
    This command supports the following options: -catalogLocation, -defaultSourceLocation, -updateSeverity,
        -updateType, -updateDeviceCategory, -report, -silent, -outputLog

    /applyUpdates - Applies all updates for the current system configuration.
    This command supports the following options: -catalogLocation, -defaultSourceLocation, -updateSeverity,
        -updateType, -updateDeviceCategory, -reboot, -silent, -outputLog, -autoSuspendBitLocker, -encryptionKey,
        -encryptedPassword, -encryptedPasswordFile, -forceUpdate

    /driverInstall - Installs all base drivers for the current system configuration on a freshly installed
        operating system.
    This command supports the following options: -driverLibraryLocation, -reboot, -silent, -outputLog

    /configure - Allows configuration of Dell Command | Update based on settings provided in the supported
        options.
    This command supports the following options: -importSettings, -exportSettings, -lockSettings,
        -biosPassword,-secureBiosPassword, -advancedDriverRestore, -driverLibraryLocation, -catalogLocation,
        -defaultSourceLocation, -allowXML, -updatesNotification, -downloadLocation, -updateSeverity,
        -updateType, -updateDeviceCategory, -userConsent, -customProxy, -proxyAuthentication, -proxyHost,
        -proxyPort, -proxyFallbackToDirectConnection, -proxyUsername, -proxyPassword,-secureProxyPassword,
        -silent, -outputLog, -scheduleAction, -scheduleDaily, -scheduleWeekly, -scheduleMonthly,
        -scheduleManual, -scheduleAuto, -systemRestartDeferral -deferralRestartInterval -deferralRestartCount,
        -installationDeferral -deferralInstallInterval -deferralInstallCount, -restoreDefaults,
        -autoSuspendBitLocker, -maxRetry, -delayDays, -forcerestart

    /generateEncryptedPassword - Generates an encrypted BIOS password.
    This command supports the following options: -encryptionKey,-secureEncryptionKey,
        -password,-securePassword, -outputPath

    /customnotification - Allows configuration of custom notifications
    This command supports the following options: -heading, -body , -timestamp. NOTE: For customnotification,
        all the three options -heading, -body, -timestamp are mandatory

    /version - Displays the version.


The following options are available to use with the commands:

    -help - Displays usage information.
    -? - Displays usage information.

    -report - Allows the user to create an XML report of the applicable updates.
    Expected value(s): <folder path>
    Ex: > dcu-cli /scan -report=C:\Temp\

    -silent - Allows the user to hide status and progress information on the console.
    Expected value(s): None
    Ex: > dcu-cli /scan -silent

    -outputLog - Allows the user to log the status and progress information of a command execution in a
        given log path.
    Expected value(s): <file path> with .log extension
    Ex: > dcu-cli /scan -outputLog=C:\Temp\scanOutput.log

    -reboot - Reboot the system automatically (if required).
    Expected value(s): <enable|disable>
    Ex:> dcu-cli /applyUpdates -reboot=enable

    -forceUpdate - Set the forceUpdate option to override pause
    Expected value(s): enable/disable
    Ex: > dcu-cli /applyupdates -forceUpdate=enable/disable

    -catalogLocation - Allows the user to set the repository/catalog file location. If used with
        /applyUpdates, only one path may be specified.
    Expected value(s): <file path(s)>
    Ex: > dcu-cli /configure -catalogLocation=C:\catalog.xml;C:\catalog2.cab
    Ex: > dcu-cli /applyUpdates -catalogLocation=C:\catalog.xml;C:\catalog2.cab

    -driverLibraryLocation - Allows the user to set the system driver catalog location. If this option is
        not specified, the driver library shall be downloaded from Dell.com. NOTE: Requires functional
        networking components.
    Expected value(s): <file path>
    Ex: > dcu-cli /configure -driverLibraryLocation=C:\Temp\DriverLibrary.cab

    -advancedDriverRestore - Allows the user to enable or disable the Advanced Driver Restore feature in the
        UI.
    Expected value(s): <enable|disable>
    Ex: > dcu-cli /configure -advancedDriverRestore=disable

    -updateSeverity - Allows the user to filter updates based on severity.
    Expected value(s): (security,critical,recommended,optional)
    Ex: > dcu-cli /configure -updateSeverity=recommended,optional

    -updateType - Allows the user to filter updates based on update type.
    Expected value(s): (bios,firmware,driver,application,utility,others)
    Ex: > dcu-cli /configure -updateType=bios

    -updateDeviceCategory - Allows the user to filter updates based on device type.
    Expected value(s): (audio,video,network,storage,input,chipset,others)
    Ex: > dcu-cli /configure -updateDeviceCategory=network,storage

    -importSettings - Allows the user to import application settings file. NOTE: This option cannot be used
        with any other options except -outputLog and -silent.
    Expected value(s): <file path>
    Ex: > dcu-cli /configure -importSettings=C:\Temp\Settings.xml

    -lockSettings - Allows the user to lock all the settings in the UI. NOTE: This option cannot be used
        with any other options except -outputLog and -silent.
    Expected value(s): <enable|disable>
    Ex: > dcu-cli /configure -lockSettings=enable

    -exportSettings - Allows the user to export application settings to the specified folder path. NOTE:
        This option cannot be used with any other options except -outputLog and -silent.
    Expected value(s): <folder path>
    Ex: > dcu-cli /configure -exportSettings=C:\Temp\

    -userConsent - Allows the user to opt-in or opt-out to send information to Dell regarding the update
        experience.
    Expected value(s): <enable|disable>
    Ex: > dcu-cli /configure -userConsent=disable

    -biosPassword - Allows the user to provide the unencrypted BIOS password. The password will be cleared
        if a password is not provided or "" is supplied. NOTE: The value needs to be enclosed in double quotes.
    Expected value(s): <password|"">
    Ex: > dcu-cli /configure -biosPassword="Test1234"

    -secureBiosPassword - Allows the user to provide the secured BIOS password which is masked when entered
        on the input prompt. The password will be cleared if a password is not provided or "" is supplied. NOTE:
        The value needs to be enclosed in double quotes.
    Expected value(s): <password|"">
    Ex: > dcu-cli /configure -secureBiosPassword
    Enter the password:"Test1234"- Displayed as ********

    -downloadLocation - Allows the user to specify the location to override the default application download
        path.
    Expected value(s): <folder path>
    Ex: > dcu-cli /configure -downloadLocation=C:\Temp\AppDownload

    -customProxy - Allows the user to enable or disable the use of custom proxy. NOTE: Setting this option
        to enable will cause validation of all custom proxy settings.
    Expected value(s): <enable|disable>
    Ex: > dcu-cli /configure -customProxy=enable

    -proxyHost - Allows the user to specify the proxy host. Giving an empty string as the value to this
        option clears proxy host. NOTE: Changing this option will cause validation of all custom proxy settings.
    Expected value(s): <FQDN|IP address|"">
    Ex: > dcu-cli /configure -proxyHost=proxy.com
    Ex: > dcu-cli /configure -proxyHost=""

    -proxyPort - Allows the user to specify the proxy port. Giving an empty string as the value to this
        option clears proxy port. NOTE: Changing this option will cause validation of all custom proxy settings.
    Expected value(s): <port|"">
    Ex: > dcu-cli /configure -proxyPort=8080
    Ex: > dcu-cli /configure -proxyPort=""

    -proxyFallbackToDirectConnection - Allows the user to enable or disable the use of internet connection
        if proxy fails. NOTE: Changing this option will cause validation of all custom proxy settings.
    Expected value(s): <enable|disable>
    Ex: > dcu-cli /configure -proxyFallbackToDirectConnection=enable

    -proxyAuthentication - Allows the user to enable or disable the use of proxy authentication. NOTE:
        Changing this option will cause validation of all custom proxy settings.
    Expected value(s): <enable|disable>
    Ex: > dcu-cli /configure -proxyAuthentication=enable

    -proxyUsername - Allows the user to specify the proxy username. Giving an empty string as the value to
        this option clears proxy username. NOTE: Changing this option will cause validation of all custom proxy
        settings.
    Expected value(s): <username|"">
    Ex: > dcu-cli /configure -proxyUsername="john doe"
    Ex: > dcu-cli /configure -proxyUsername=""

    -proxyPassword - Allows the user to specify the proxy password. Giving an empty string as the value to
        this option clears proxy password. NOTE: Changing this option will cause validation of all custom proxy
        settings. The value needs to be enclosed in double quotes
    Expected value(s): <password|"">
    Ex: > dcu-cli /configure -proxyPassword="my password"
    Ex: > dcu-cli /configure -proxyPassword=""

    -secureProxyPassword - Allows the user to specify the proxy password. Giving an empty string as the
        value to this option clears proxy password. NOTE: Changing this option will cause validation of all
        custom proxy settings. The value needs to be enclosed in double quotes
    Expected value(s): <password|"">
    Ex: > dcu-cli /configure -secureProxyPassword
    Ex: > Enter the password:"my password"- Displayed as ********
    Ex: > Enter the password:""- Displayed as *

    -scheduleAction - Allows the user to specify the action to perform when updates are found.
    Expected value(s): <NotifyAvailableUpdates | DownloadAndNotify | DownloadInstallAndNotify>
    Ex: > dcu-cli /configure -scheduleAction=NotifyAvailableUpdates

    -scheduleDaily - Allows the user to specify the time on which to schedule an update. NOTE: This option
        cannot be used with -scheduleManual, -scheduleAuto, -scheduleMonthly, -scheduleWeekly.
    Expected value(s): Time[00:00(24 hr format, 15 mins increment)]
    Ex: > dcu-cli /configure -scheduleDaily=23:45

    -scheduleWeekly - Allows the user to specify the day of the week and time on which to schedule an
        update. NOTE: This option cannot be used with -scheduleManual, -scheduleAuto, -scheduleMonthly,
        -scheduleDaily.
    Expected value(s): Day [< Sun | Mon | Tue | Wed | Thu | Fri | Sat >],Time[00:00(24 hr format, 15 mins
        increment)]
    Ex: > dcu-cli /configure -scheduleWeekly=Mon,23:45

    -scheduleMonthly - Allows the user to specify schedule values in two format to schedule an update. First
        format allows user to specify the day of the month and time. Second format allows user to specify the
        week, day and time of month. If the scheduled day is greater than the last day of the month, the update
        will be performed on the last day of that month. NOTE: This option cannot be used with -scheduleManual,
        -scheduleAuto, -scheduleWeekly, -scheduleDaily
    Expected value(s) for first format: Date of month [1 - 31],Time[00:00(24 hr format, 15 mins increment)]
    Expected value(s) for second format: Week [< first | second | third | fourth | last >],Day [< Sun | Mon
        | Tue | Wed | Thu | Fri | Sat >],Time[00:00(24 hr format, 15 mins increment)]
    Ex: > dcu-cli /configure -scheduleMonthly=28,00:45
    Ex: > dcu-cli /configure -scheduleMonthly=second,Fri,00:45

    -scheduleManual - Allows the user to disable the automatic schedule and enable only manual updates.
        NOTE: This option cannot be used with -scheduleAuto, -scheduleWeekly, -scheduleMonthly, -scheduleDaily
    Expected value(s): None
    Ex: > dcu-cli /configure -scheduleManual

    -scheduleAuto - Allows the user to enable the default automatic update schedule. NOTE: Automatic updates
        execute every 3 days. Also, this option cannot be used with -scheduleManual, -scheduleWeekly,
        -scheduleMonthly, -scheduleDaily
    Expected value(s): None
    Ex: > dcu-cli /configure -scheduleAuto

    -restoreDefaults - Allows the user to restore default settings.
    Expected value(s): None
    Ex: > dcu-cli /configure -restoreDefaults

    -autoSuspendBitLocker - Allows the user to enable or disable the automatic suspension of BitLocker, when
        applying BIOS updates.
    Expected value(s): <enable|disable>
    Ex: > dcu-cli /configure -autoSuspendBitLocker=enable

    -defaultSourceLocation - Enable or Disable the default source location (dell.com).
    Expected value(s): <enable|disable>
    Ex: > dcu-cli /configure -defaultSourceLocation=enable

    -allowXML - Allows the user to Enable or Disable XML Catalog file.
    Expected value(s): <enable|disable>
    Ex: > dcu-cli /configure -allowXML=enable
    Ex: > dcu-cli /configure -allowXML=enable -catalogLocation=C:\catalog.xml
    Ex: > dcu-cli /configure -catalogLocation=C:\catalog.xml -allowXML=enable

    -updatesNotification - Enable or Disable toast Notifications
    Expected value(s): <enable|disable>
    Ex: > dcu-cli /configure -updatesNotification=enable

    -systemRestartDeferral - Allows the user to Enable or Disable system restart deferral options. Note:
        -deferralRestartInterval and -deferralRestartCount is required to be specified along with Enable option.
    Expected value(s): <enable|disable>
    -deferralRestartInterval -Allows the user to set the deferral restart interval
    Expected value(s):  <1-99>
    -deferralRestartCount -Allows the user to set the deferral restart count
    Expected value(s):  <1-9>
    Ex: > dcu-cli /configure -systemRestartDeferral=enable -deferralRestartInterval=1
        -deferralRestartCount=2
    Ex: > dcu-cli /configure -systemRestartDeferral=disable

    -installationDeferral - Allows the user to Enable or Disable deferral install options. Note:
        -deferralInstallInterval and -deferralInstallCount is required to be specified along with Enable option.
    Expected value(s): <enable|disable>
    -deferralInstallInterval -Allows the user to set the deferral install interval
    Expected value(s):  <1-99>
    -deferralInstallCount -Allows the user to set the deferral install count
    Expected value(s):  <1-9>
    Ex: > dcu-cli.exe /configure -installationDeferral=enable -deferralInstallInterval=1
        -deferralInstallCount=2
    Ex: > dcu-cli.exe /configure -installationDeferral=disable

    -maxRetry - Set the maximum retry attempts to install failed updates upon reboot. By default, the value
        is set to 2.
    Expected value(s): <1|2|3>
    Ex: > dcu-cli /configure -maxRetry=2

    -delayDays - Allow users to set the delay for updates.. By default, the value is set to 0.
    Expected value(s): 0 to 45
    Ex: > dcu-cli /configure -delayDays=2

    -forceRestart - Allows the user to Enable or Disable Force Restart during scheduled operation.
    Expected value(s): <enable|disable>
    Ex: > dcu-cli /configure -forceRestart=enable

    -encryptedPassword - Allows the user to pass the encrypted password in-line along with the encryption
        key that was used to generate it. NOTE: -encryptionKey is required to be specified along with this
        option. Also, this value needs to be enclosed in double quotes.
    Expected value(s): <encrypted password>
    Ex: > dcu-cli /applyUpdates -encryptedPassword="myEncryptedPassword" -encryptionKey="myEncryptionKey"

    -secureEncryptedPassword - Allows the user to pass the encrypted password on input prompt along with the
        secure encryption key that was used to generate it. NOTE: -securedEncryptionKey is required to be
        specified along with this option. Also, this value needs to be enclosed in double quotes.
    Expected value(s): <secure encrypted password>
    Ex: > dcu-cli /applyUpdates -secureEncryptedPassword -secureEncryptionKey
    Ex: > Enter the password:"myEncryptedPassword"- Displayed as *******************
    Ex: > Enter the encryptionkey:"myEncryptionKey"- Displayed as ***************

    -encryptionKey - Allows the user to specify the encryption key used to encrypt the password. NOTE: The
        key provided must be at least six nonwhitespace characters and include an uppercase letter, a lowercase
        letter, and a digit. Also, this value needs to be enclosed in double quotes.
    Expected value(s): <encryption key>
    Ex: > dcu-cli /applyUpdates -encryptedPassword="myEncryptedPassword" -encryptionKey="myEncryptionKey"
    Ex: > dcu-cli /generateEncryptedPassword -encryptionKey="myEncryptionKey" -password="myPassword"
        -outputPath=C:\Temp

    -secureEncryptionKey - Allows the user to specify the secured encryption key used to encrypt the secured
        password. NOTE: The key provided must be at least six nonwhitespace characters and include an uppercase
        letter, a lowercase letter, and a digit. Also, this value needs to be enclosed in double quotes.
    Expected value(s): <secure encryption key>
    Ex: > dcu-cli /applyUpdates -secureEncryptedPassword -secureEncryptionKey
    Ex: > Enter the password:"myEncryptedPassword"- Displayed as *******************
    Ex: > Enter the encryptionkey:"myEncryptionKey"- Displayed as ***************
    Ex: > dcu-cli /generateEncryptedPassword -secureEncryptionKey -securePassword -outputPath=C:\Temp
    Ex: > Enter the encryptionkey:"myEncryptionKey"- Displayed as ***************
    Ex: > Enter the password:"myEncryptedPassword"- Displayed as *******************

    -encryptedPasswordFile - Allows the user to pass the encrypted password via file. NOTE: -encryptionKey
        is required to be specified along with this option.
    Expected value(s): <file path>
    Ex: > dcu-cli /applyUpdates -encryptedPasswordFile=C:\Temp\encryptedPassword.txt
        -encryptionKey="myEncryptionKey"

    -password - Allows the user to specify the password to be encrypted. NOTE: -encryptionKey is required to
        be specified along with this option. Also, this value needs to be enclosed in double quotes.
    Expected value(s): <password>
    Ex: > dcu-cli /generateEncryptedPassword -encryptionKey="myEncryptionKey" -password="myPassword"

    -securePassword - Allows the user to input the secured password to be encrypted. NOTE:
        -secureEncryptionKey is required to be specified along with this option. Also, this value needs to be
        enclosed in double quotes.
    Expected value(s): <securePassword>
    Ex: > dcu-cli /generateEncryptedPassword -secureEncryptionKey -securePassword
    Ex: > Enter the encryptionkey:"myEncryptionKey"- Displayed as ***************
    Ex: > Enter the password:"myEncryptedPassword"- Displayed as *******************

    -outputPath - Allows the user to specify the folder path to which encrypted password file is saved.
    Expected value(s): <folder path>
    Ex: > dcu-cli /generateEncryptedPassword -encryptionKey="myEncryptionKey" -password="myPassword"
        -outputPath=C:\Temp

    -heading -Allows the user to set the heading for the notification.
    Expected value(s): text for heading of the notification.The maximum length of the heading is 80
        characters.
    -body -Allows the user to set the content/body for the notification.
    Expected value(s): text for content of the notification.The maximum length of the body is 750
        characters.
    -timestamp -Allows the user to set the timestamp for the notification.
    Expected value(s): future date and time to schedule the notification.
    Ex: > dcu-cli.exe /customnotification -heading="I am heading" -body="I am body"
        -timestamp=9/19/2022,00:46

Note: The folders listed below are reserved for system use and are restricted for user level access:
    C:\Windows
    C:\Program Files
    C:\Program Files (x86)
    C:\Users\Public
    C:\ProgramData
    C:\ProgramData\UpdateService\Clients

Note: C:\ProgramData and all it's subfolders are restricted for user level access except for Dell
subfolder - C:\ProgramData\Dell

    The sub-folders, "Microsoft" and "Windows", under the following System folders are restricted for user
        level access:
    C:\Users\<UserName>\AppData\Roaming
    C:\Users\<UserName>\AppData\Local
    C:\Users\<UserName>

    Application logs (files with extension ".log") can be stored under C:\ProgramData\Dell.


The program exited with return code: 0
```
