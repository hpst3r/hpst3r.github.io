---
title: "Installing Ninja Remote (NinjaRMM remote session utility) on Linux with Wine"
date: 2025-04-26T18:30:00-00:00
draft: false
---

NinjaRMM sucks. Here's how to run its terrible little connection tool on a proper OS.

First, install a user-agent spoofer in your preferred browser - Ninja's site won't even show the connection button without this nowadays. Set it to Windows something.

Here are links to a user agent spoofer-manager thing:
- [for Chrome](https://chromewebstore.google.com/detail/user-agent-switcher-and-m/bhchdcejhohfmigjafbampogmaanbfkg)
- [for Firefox](https://addons.mozilla.org/en-US/firefox/addon/user-agent-string-switcher/)

Here are links to the most recent user agent strings:
- [for Edge](https://www.whatismybrowser.com/guides/the-latest-user-agent/edge)
- [for Chrome](https://www.whatismybrowser.com/guides/the-latest-user-agent/chrome)
- [for Firefox](https://www.whatismybrowser.com/guides/the-latest-user-agent/firefox)

You're after the Windows versions. In my case, I set my user agent string to "`Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/135.0.0.0 Safari/537.36 Edg/135.0.3179.98`".

You can confirm that everything is working with the wonderful [whatismybrowser.com](https://www.whatismybrowser.com).

{{< figure src="images/whatismybrowser-edge-firefox-spoofed-linux.png" >}}

You'll need Wine. If you don't have Wine, you can typically install it with `dnf install wine` or `apt install wine`.

To download the NinjaRemote installer, navigate to a device in Ninja. If you've done things correctly, you'll see the Ninja Remote 'connect' button. Click the button. Wait for the connection to time out, and download the 32-bit version of NinjaRemote from the pop-up.

Open up a terminal. To install NinjaRemote, run the executable with Wine, then click through the applet:

```sh
$ wine ~/Downloads/ncinstaller-x86.exe
```

Create a .desktop file for Ninja Remote:

```sh
$ cat <<EOT >> ~/.local/share/applications/ninja-remote.desktop
[Desktop Entry]
Name=Ninja Remote
Exec=bash -c 'wine "/home/liam/.wine/drive_c/users/liam/AppData/Roaming/NinjaRemote/ncplayer.exe" "%u"'
Type=Application
Terminal=false
MimeType=x-scheme-handler/ninjarmm
EOT
```

Register the .desktop file with your shell:

```sh
$ xdg-desktop-menu install ~/.local/share/applications/ninja-remote.desktop
```

Register the .desktop file as a handler for NinjaRMM custom protocol URLs:

```sh
$ xdg-mime default ninja-remote.desktop x-scheme-handler/ninjarmm
```

All in one:

```sh
wine ~/Downloads/ncinstaller-x86.exe

cat <<EOT >> ~/.local/share/applications/ninja-remote.desktop
[Desktop Entry]
Name=Ninja Remote
Exec=bash -c 'wine "/home/liam/.wine/drive_c/users/liam/AppData/Roaming/NinjaRemote/ncplayer.exe" "%u"'
Type=Application
Terminal=false
MimeType=x-scheme-handler/ninjarmm
EOT

xdg-desktop-menu install ~/.local/share/applications/ninja-remote.desktop

xdg-mime default ninja-remote.desktop x-scheme-handler/ninjarmm
```

You should now be able to Ninja Remote to PCs from your Linux box as you normally would on a Windows PC.
