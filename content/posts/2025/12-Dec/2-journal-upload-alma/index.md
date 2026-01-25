---
title: "systemd-journal-upload on AlmaLinux"
date: 2025-12-08T01:00:00-00:00
draft: false
---

See [systemd-journal-remote(8)](https://www.freedesktop.org/software/systemd/man/latest/systemd-journal-remote.service.html), [journal-remote.conf(5)](https://www.freedesktop.org/software/systemd/man/latest/journal-remote.conf.html) for detailed setup instructions. Consider this a minimal config example. [See this in my NixOS configuration](https://github.com/hpst3r/nixos/blob/b963e38c75ae8c80320dfd9770e0691422317b20/modules/services/monitoring/agents/journald-upload.nix).

## AlmaLinux 10

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

Add an override to the default service config so `systemd-journal-upload` doesn't give up after five tries (the default).

```sh
sudo tee /etc/systemd/system/systemd-journal-upload.service.d/override.conf > /dev/null << 'EOT'
[Service]
Restart=always
EOT
```

Enable the service:

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now systemd-journal-upload
```

## Proxmox VE 9

Install the `systemd-journal-remote` package:

```sh
sudo apt install -y systemd-journal-remote
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

Add an override to the default service config so `systemd-journal-upload` doesn't give up after five tries (the default).

```sh
sudo mkdir -p /etc/systemd/system/systemd-journal-upload.service.d
sudo tee /etc/systemd/system/systemd-journal-upload.service.d/override.conf > /dev/null << 'EOT'
[Service]
Restart=always
EOT
```

Enable the service:

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now systemd-journal-upload
```
