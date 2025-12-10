---
title: "systemd-journal-upload on AlmaLinux"
date: 2025-12-08T01:00:00-00:00
draft: false
---

AlmaLinux 10.1 6.12.0 on amd64 (host "3060t0") with systemd 257-13.el10

See [systemd-journal-remote(8)](https://www.freedesktop.org/software/systemd/man/latest/systemd-journal-remote.service.html), [journal-remote.conf(5)](https://www.freedesktop.org/software/systemd/man/latest/journal-remote.conf.html) for detailed setup instructions. Consider this a minimal config example. [See this in my NixOS configuration](https://github.com/hpst3r/nixos/blob/b963e38c75ae8c80320dfd9770e0691422317b20/modules/services/monitoring/agents/journald-upload.nix).

Install the `systemd-journal-remote` package:

```sh
sudo dnf install -y systemd-journal-remote
```

Populate the configuration file. Adjust to fit. I'm pulling all of my logs into a VictoriaLogs instance; you can use another machine running `journal-remote` or any other log server that takes the journal-upload format.

If you intend to use port 443, you'll need to modify SELinux policy. I've made my VictoriaLogs server listen on the default port (19532) for systemd-journal-upload instead.

```sh
sudo tee /etc/systemd/journal-upload.conf > /dev/null << 'EOT'
[Upload]
# Compression = zstd:3 # not supported until systemd 258, n/a for alma 10
# NetworkTimeoutSec is unset
ServerCertificateFile = -
ServerKeyFile = -
TrustedCertificateFile = -
URL = https://victorialogs.lab.wporter.org:19532/insert/journald
EOT
```

Enable the service:

```sh
sudo systemctl enable --now systemd-journal-upload
```
