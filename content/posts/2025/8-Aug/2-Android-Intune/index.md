---
title: "Notes on managing Android devices with Microsoft Intune"
date: 2025-08-09T17:30:00-00:00
draft: false
---

This isn't super polished, but should give you a good idea of how using Intune to manage Android devices works if you've dealt with Windows or iOS management in the past. Notes made while prepping for the Microsoft MD-102.

## Prerequisites

### Managed Google Play account setup

Before getting started, we'll need to get a managed Google Play account set up.

Navigate to the Intune admin center (intune.microsoft.com), then click on Devices > Android > Enrollment. Select the "Managed Google Play" option under "Prerequisites":

{{< figure src="images/0-gplay-enrollment-prereq.png" >}}

In the "Managed Google Play" blade, check the "I agree" box (under 'I grant Microsoft permission to send user and device information to Google'), then click 'connect now'.

{{< figure src="images/1-gplay-connect.png" >}}

Create a Google admin account (preferably using your work email address), then link Android Enterprise to Intune:

{{< figure src="images/2-link-gplay.png" >}}

Once the window closes, the Managed Google Play blade will show a nice, happy check mark:

{{< figure src="images/3-gplay-done.png" >}}

### Configure the Intune Provisioning Client service principal to add devices to groups when joined

As with any Intune devices, you'll probably want to assign your devices to a specific group. This will be a normal Entra ID group that has the Intune Provisioning Client service principal registered as an Owner.

It may be necessary to create the Intune Provisioning Client service principal. [See Microsoft documentation for this topic here](https://learn.microsoft.com/en-us/autopilot/device-preparation/tutorial/user-driven/entra-join-device-group#adding-the-intune-provisioning-client-service-principal).

The principal has ID f1346770-5b25-470b-88bd-d5744ab7952c and may be named "Intune Autopilot ConfidentialClient" or "Intune Provisioning Client" depending on the age of the tenant.

## Methods of device enrollment

I'll be using a physical Pixel 6a for all sections here. At the time of writing it's a supported and "Enterprise Recommended" phone. Mine's running Android 16 Beta.

Any Android device that runs Android 8 or later and has Google Mobile Services should work, including an Android Studio emulated device (though, as you can't wipe an Android Studio device, you won't be able to easily get a device fully managed).

Only some devices support Android Enterprise (mostly Google, Samsung and Motorola traditional phones, plus a number of rugged devices). A full list of supported devices can be found at [android.com/enterprise/devices](https://www.android.com/enterprise/devices/).

There are a few ways that Intune will let you enroll Android devices:

- Personally owned, with a work profile
  - BYOD, but work apps are sandboxed and IT admins can configure DLP, encryption at rest, etc in this container, or wipe it entirely when the employee leaves. The user can easily schedule their work apps or pause them manually when they don't want to receive notifications. This is a very slick option and works well.
- Corporate owned, dedicated devices
  - Kiosks and "task" (e.g., inventory management) devices without an assigned owner. This is just a normal managed device without an associated account. Fully managed phone, allows IT to block installation of apps, prevent users from removing apps, and prevent users from resetting devices.
- Corporate-owned, fully managed devices
  - Corporate owned devices with one profile - the work profile. Users sign in with their Microsoft credentials and get SSO to apps on the device. Fully managed phone, allows IT to block installation of apps, prevent users from removing apps, and prevent users from resetting devices.
- Corporate-owned, with a work profile
  - Corporate-owned devices with a separate 'work' and 'personal' profile, so users can install apps. You would use this only if you allow work phones to be used for personal use. Unfortunately, this means you cannot see the phone number of the device from MDM - otherwise, it works nearly identically to a personally-owned device with a work profile.

### Personally owned devices with a work profile

A 'work profile' is a sandboxed environment on an Android phone that allows you, the IT admin, to force users to separate their work and personal data on a BYOD phone. Android 'work profiles' provide distinct separation between personal and work data:

{{< figure src="images/4-WorkApps.png" width="350" >}}

And can prevent your users from copying files off their phone, or forwarding a screenshot of Teams over to a friend:

{{< figure src="images/5-ExfilWorkFiles.png" width="350" >}}

Let's walk through the lifecycle of a work profile!

#### Setup experience (Enrollment)

Users will have to install Company Portal, sign in with a Microsoft account, then agree to allow your organization to manage a profile on their devices:

{{< figure src="images/6-CompanyPortal1.png" width="350" >}}

{{< figure src="images/7-CompanyPortal2.png" width="350" >}}

{{< figure src="images/8-CompanyPortal3.png" width="350" >}}

{{< figure src="images/9-DownloadingWorkProfile.png" width="350" >}}

{{< figure src="images/10-DownloadingWorkProfile2.png" width="350" >}}

{{< figure src="images/11-DownloadingWorkProfile3.png" width="350" >}}

If the device is noncompliant (per your policy - e.g., rooted, failing Google Play API integrity check) the user will be notified (and asked to contact support for assistance) at this stage, and enrollment will not continue.

If the device *is* compliant, the user will then be able to open the managed Google Play app and download applications from the organization for their work profile. They'll be notified of this when enrollment completes:

{{< figure src="images/12-company-portal-regi-complete.png" width="350" >}}

Users will have the option to pause work apps (either manually, or on a schedule) so they don't get notifications from work outside of work hours. As far as I can tell, there is no way to hide this, but it does seem to work quite well.

### Apps

"Required" applications will be installed in the work profile without further user interaction when the device registers. Applications that are "Available for enrolled devices" or "Available with or without enrollment" will be available from the Managed Play Store (denoted by a briefcase icon, and appearing in the Work tab of the app drawer).

{{< figure src="images/13-managed-gplay.png" width="350" >}}

On a personal or company owned device with a work profile, the user can remove Required applications, though they are reinstalled by Intune the next time the device checks in with the MDM service.

Microsoft apps in the Work profile can take advantage of (nearly) seamless single sign-on - the work profile will be registered with Entra (not joined) when the profile is created.

### Protect

If the administrator configures encryption, DLP, or a managed browser, this will affect anything in the work profile. I believe the default settings I pushed to my device restrict copy/paste, screenshots, and file transfers from the work profile to the personal profile fairly effectively - I could not quickly find a way around this (other than sending an email out via my Outlook client, in the work profile, that would be subject to DLP if configured properly.)

A Remote Lock on a device will lock the entire device, notify the user that the device was locked by 'work policy', disable biometrics (require the device's primary PIN) *and* require the user to unlock the work profile with its (potentially separate) PIN. This is subject to the same six hour refresh period as any other Intune action.

A passcode reset will reset the passcode for the work profile to a generated string (displayed in a banner in the Intune admin center for a week post-reset). The user will be notified that they need to reauthenticate to gain access to their work profile, and biometrics for device unlock will be temporarily disabled (as with a Remote Lock).

It does not seem that the 'Reset password' action will prompt the user to set their password - they'll have to go find this option and manually initiate it. This might be an annoyance considering the length of the temporary passphrase (see below).

{{< figure src="images/14-intune-byod-android.png" >}}

You can configure Protection policies that will, e.g., enforce encryption at rest or wipe data from apps after a set number of days without contact to Intune at Apps > Android > Protection.

You can configure policy for apps, e.g., Edge extension forcelist and which URLs are opened when a browser launches under Apps > Android > Configuration. Most things do not appear to have a settings catalog on Android.

### Retire

When the employee has a new phone or a new employer, or their phone has a new thief, it's probably time to retire or delete their work profile.

You cannot wipe your employees' entire phone (unfortunately).You *can* spam them with notifications or nuke their work profile, though. When the user has been terminated, the device is lost, or if the device has not checked in for a period of time, you can manually or automatically delete the work profile (and thus make the device noncompliant, removing all company data).

This is likely a separate action from disabling the account (though the user's work profile will quickly become useless if you disable their Microsoft account), but I don't feel like signing in again everywhere after I've blocked myself to test it out.

To manually delete the work profile from a phone, you can select either 'Delete' or 'Retire'. A 'Delete' action will immediately remove the device from the Intune admin center, while a 'Retire' will not. Otherwise, these are identical.

If you retire the device, it'll be deleted from Intune when it checks in next (at which point it'll delete the work profile and return itself to a boring old personal phone).

If the work profile is encrypted, your data is theoretically not accessible to a bad actor who grabs the phone, and if they connect the device to the Internet it'll delete itself entirely (assuming there's a delete pending). Assuming you've configured a Protection Policy, your data should be wiped after a set period without connectivity to the MDM service.

{{< figure src="images/15-android-device-delete-retire.png" >}}

To automatically remove stale/inactive devices after a set period, you can configure a clean-up rule. This will immediately begin to purge devices that have not checked in during a set interval.

{{< figure src="images/16-cleanup-rule.png" >}}

#### Block legacy enrollment

You may want to block deprecated Android device administrator Intune enrollment. You can do this by creating an Enrollment Restriction (under Intune > Devices > Enrollment > Personally owned devices with work profile > Enrollment restrictions) and targeting a group, or editing the default Enrollment Restriction to block this for all users:

{{< figure src="images/17-block-ada.png" >}}

### Corporate-owned, fully managed or dedicated devices

Enrolling devices in the Android OOBE or via Zero Touch will let you turn them into fully-managed corporate devices. This is more like the sole profile that Apple iOS devices have when managed via Microsoft Intune - completely locked down for corporate use.

To create an enrollment token for manual enrollment, navigate to Intune > Devices > Android > Enrollment > Corporate-owned, fully managed user devices (or dedicated devices - same process):

{{< figure src="images/18-android-enrollment.png" >}}

Create a policy and generate a token:

{{< figure src="images/19-enrollment-token.png" >}}

This is the only thing you'll need to enroll devices (once Managed Google Play is configured, that is). Now, we can begin the device lifecycle.

### Lifecycle

The device lifecycle of a fully managed Android is extremely similar to that of a 'work profile', with the exception of enrollment.

#### Enroll

With zero-touch enrollment, the device will be ready as soon as it's taken out of the box. Otherwise, to manually enroll a device from the out-of-box experience, tap the first screen five times, then scan the token you created (QR code).

{{< figure src="images/20-oobe-scanning-qr-cropped.jpg" width="350" >}}

Connect the device to WiFi, let it provision itself. If this is a user device, not a dedicated device, the user will be prompted to sign in with their Microsoft credentials:

{{< figure src="images/21-oobe-entra-sign-in-cropped.jpg" width="350" >}}

The device will be Entra registered when the user signs in - this enables (nearly) seamless SSO to Microsoft applications on the device immediately post-setup. The provisioning process will also automatically configure the Authenticator app on the device for MFA, which is a nice touch.

If the device is dedicated, the user will not be prompted for credentials. The out-of-box experience will guide the user through configuring a PIN and biometrics (if configured), install apps, then guide the user through gesture navigation mode (you can probably block this bit, but I didn't look for the option).

Regardless, once the device has exited the OOBE, it'll be ready for use (no Google account required!)

If you've configured a group assignment for the token, the devices will be dynamically joined to said group and will receive apps targeted as "required" as soon as the OOBE is completed.

Dedicated devices, without a specific user, will obviously not receive apps targeted by user - you must target devices.

#### Configure

As with a work profile on a personal phone, on a fully-managed phone, you can get apps via the Managed Google Play store if they're made available, or if access to the entire Play store is enabled.

Unlike a personal device, **fully-managed** corporate Android Enterprise devices do not have an unmanaged Play store - they either get a normal-looking Play Store with a tab for work apps, or just the locked-down approved-app-only Play Store. There's no separate work and personal profile here, obviously:

{{< figure src="images/22-fully-managed-drawer.png" width="350" >}}

Required apps will be installed when the device exits the OOBE, and appear as 'Installed' in Google Play. They cannot be uninstalled from a managed device.

{{< figure src="images/23-fully-managed-cannot-uninstall-required.png" width="350" >}}

If full access to the Play store is not enabled, only apps made available by IT admins are available, and Google Play does not show the normal experience (though users still go to Google Play for their apps, rather than the Company Portal - this is a nice UX improvement over the managed iOS experience).

{{< figure src="images/24-fully-managed-restricted-play-store.png" width="350" >}}

If full access *is* enabled, there's a "Work Apps" section in the managed Play store (third tab from For You on the left).

If you *are* using any of the Fully Managed modes, I would probably recommend taking advantage of Common Criteria mode (Configuration > System Security > Require Common Criteria mode) to immediately harden your devices with minimal (to no) user impact. [See details here](https://www.commoncriteriaportal.org/files/ppfiles/pp_md_v3.1.pdf).

When configuring policies for Android Enterprise devices, you *must* use a settings template or directly specify CSP paths to get access to the majority of the Intune configuration options - for whatever reason, the settings catalog is nearly blank.

Control over fully-managed Androids via Intune is very thorough. For an example:

{{< figure src="images/25-intune-device-configuration.png" >}}

{{< figure src="images/26-intune-device-configuration-2.png" >}}

The settings you probably want are:

- Blocking personal Google accounts entirely (they probably aren't needed at all if you're wholly in the Microsoft ecosystem)
- Allowing or blocking (default) the Play store on managed devices (you can allow full access without a Google account, which is slick)
- Blocking device wipe
- Disallow adding users
- Blocking file transfer, external media, tethering, Bluetooth, NFC, etc
- Forcing automatic device updates
- Requiring Common Criteria mode (configuration baseline that hardens the device)
- Requiring a PIN and biometrics

#### Protect

As with any other mobile device, you'll probably wind up configuring policy and compliance to require a passcode, set screen lock, and configure encryption at rest.

The user will be guided through the process of setting up a PIN that meets your requirements in the OOBE and the device will handle most configuration, like encryption, on its own.

If the user attempts to use a 'protected' action, e.g., taking a screenshot when screenshots are disabled by Intune configuration, they'll be told that this feature was disabled by their IT admin and nothing will happen.

If the device goes out of compliance with policy, you should block access to corporate resources from it via Conditional Access (as with any other device managed by Intune).

#### Retire

When it comes time to retire a fully managed Android, just click 'Wipe' on its page in the Intune admin center (or send an API request to do the same). The device will, upon its next check-in, restart, wipe data, and return to the factory state.

The same wipe timeouts you can configure on personal devices will apply to fully managed Androids, but this will wipe the entire device (not just the work profile).

## Zero-touch provisioning

If you acquire devices from an OEM that supports Android Enterprise zero-touch provisioning (many do - including, of course, [Google](https://pixel.google/business/)) they can be enrolled in MDM the moment they're powered on.

Judging by how well the rest of the Android Enterprise experience works, I'd expect this to do better than Apple Business Manager MDM enrollment in the iOS OOBE (boy, is that painful). However, since I don't have a need to buy a bunch of Android phones right this moment, I won't have the chance to try this out.

## Assigning Google Play apps to Android devices

To approve apps for the Managed Play Store, open up the Play store from Intune and select the ones you'd like, then hit Sync at the top of the page, go back, and assign the apps to your groups.

Navigate to Apps > Android > Add > Managed Google Play, and you should (after hitting Sync and waiting a few moments, possibly trying again) be able to pop open the Play store in an iframe. Here, you can search for and approve Android apps.

Click Select on an app, then hit Sync in the top left. Wait a few moments, and the app you've selected should appear in the app list in Intune, where you can configure a deployment as usual (click the app, select Properties, Edit your Assignments, and select an assignment type and targets).

{{< figure src="images/27-managed-play-add-app.png" >}}

If you make an app available or required for devices or users, it'll appear in their Managed Google Play app store (either in the main view, if they've got a locked-down Play store, or in the Work tab if they have unrestricted access).
