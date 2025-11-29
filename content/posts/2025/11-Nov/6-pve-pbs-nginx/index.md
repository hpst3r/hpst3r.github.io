---
title: "Proxying the Proxmox VE & Backup Server web UIs with NGINX"
date: 2025-11-28T22:30:00-00:00
draft: false
---

See [docs (pve.proxmox.com)](https://pve.proxmox.com/wiki/Web_Interface_Via_Nginx_Proxy).

For more info on using the built-in ACME certificate renewal utilities, see [this other post on that subject (wporter.org)](proxmox-ve-backup-server-built-in-acme-cert-renewal/).

## PVE

### Simpler (maybe) - reuse the proxmox-acme script

If you want to use the built-in `/usr/share/proxmox-acme` 'requester' (managed with `pvenode` or the web UI), omit a HTTP redirect/ACME block so NGINX doesn't eat port 80 and just point NGINX at the `/etc/pve/nodes/hostname/pveproxy-ssl.pem` certificate.

I prefer this method, since it's far simpler, makes sure Proxmox is using a trusted cert, and because adding random packages to hypervisors is not one of the best things to do.

If you need to make adjustments to renewal timing in this scenario, override the `pve-daily-update.timer`systemd timer and, optionally, set the `pve-daily-update.service` oneshot to be `WantedBy=multi-user.target` via another override, e.g.:

```sh
mkdir -p /etc/systemd/system/pve-daily-update.service.d
tee /etc/systemd/system/pve-daily-update.service.d/override.conf > /dev/null << 'EOT'
[Install]
WantedBy=multi-user.target
EOT
systemctl daemon-reload
systemctl enable pve-daily-update.service
```

Anyway! Onto the good stuff.

Install NGINX, and remove the default configuration:

```sh
apt install -y nginx
rm /etc/nginx/sites-enabled/default
```

Drop in a basic NGINX configuration to proxy 443 to localhost:8006 using the Proxmox-managed SSL certificates on the pmxcfs:

```sh
tee /etc/nginx/conf.d/proxmox.conf > /dev/null << 'EOT'
upstream proxmox {
 server "127.0.0.1:8006";
}

server {
 listen 443 ssl;
 listen [::]:443 ssl;
 ssl_certificate /etc/pve/local/pveproxy-ssl.pem;
 ssl_certificate_key /etc/pve/local/pveproxy-ssl.key;
 proxy_redirect off;
 location / {
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  proxy_pass https://proxmox;
 }
}
EOT
```

Test the configuration:

```sh
nginx -t
```

If it passes, enable & start NGINX:

```sh
systemctl enable --now nginx
```

Test with cURL or your preferred flavor of Chromium:

```sh
wporter@wm3 ~ % curl -s https://800g4m.lab.wporter.org:443 | grep title
    <title>800g4m - Proxmox Virtual Environment</title>
```

Since certificates (in this case) rely on the pmxcfs, which isn't up until the `pve-cluster.service` starts, override the systemd unit for NGINX to start after the `pve-cluster.service`.

```sh
mkdir -p /etc/systemd/system/nginx.service.d
tee /etc/systemd/system/nginx.service.d/override.conf > /dev/null << 'EOT'
[Unit]
Requires=pve-cluster.service
After=pve-cluster.service

[Service]
Restart=always
RestartSec=5
StartLimitBurst=5
EOT
```

In my case, because I'm using DNS rather than the hosts file, Proxmox was starting the `pve-cluster` service early, and it was then immediately failing, taking NGINX with it. I fixed this dependency issue by kindly asking `pve-cluster` to wait for the network to come up.

Example follows. Not great, but I'm tired and want to go to sleep.

```sh
mkdir -p /etc/systemd/system/pve-cluster.service.d
tee /etc/systemd/system/pve-cluster.service.d/override.conf > /dev/null << 'EOT'
[Unit]
After=network-online.target nss-lookup.target
Wants=network-online.target

[Service]
ExecStartPre=/bin/bash -c 'for i in {1..60}; do IP=$(getent hosts 800g4m 2>/dev/null | awk "{print \$1}" | grep -v "^127\."); if [ -n "$IP" ]; then exit 0; fi; sleep 1; done; exit 1'
EOT
```

### Less simple - use Certbot and NGINX for ACME

If you would prefer to not use Proxmox's ACME utility, you can certainly use a more standard NGINX configuration with Certbot.

Note that this will break Proxmox's ACME 'requester' by preventing it from binding to port 80, if that matters in your environment (e.g., for trusted SSL to the API on port 8006).

```sh
apt install -y nginx certbot
rm /etc/nginx/sites-enabled/default
tee /etc/nginx/conf.d/proxmox.conf > /dev/null << 'EOT'
upstream proxmox {
  server 127.0.0.1:8006;
}

server {

  listen 443 ssl;
  listen [::]:443 ssl;
  
  server_name _;

  # SSL
  ssl_certificate /etc/letsencrypt/live/placeholder.domain.example/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/placeholder.domain.example/privkey.pem;
  #ssl_trusted_certificate /etc/letsencrypt/live/placeholder.domain.example/chain.pem;

  # . files
  location ~ /\.(?!well-known) {
      deny all;
  }

  location / {
    proxy_pass                  https://proxmox;
    proxy_ssl_verify            off;
    proxy_http_version          1.1;
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host       $http_host;
    proxy_set_header X-Real-IP  $remote_addr;
  }

  gzip on;
  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 6;
  gzip_types text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;

}

# http redirect
server {

    listen 80;
    listen [::]:80;
    
    # certbot
    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/letsencrypt;
        allow all;
        add_header Cache-Control "no-store";
    }
    
    location / {
        return 301 https://$host$request_uri;
    }
}
EOT
```

To replace the placeholder directory names in the example NGINX config with your server's FQDN:

```sh
domain_name='800g4m.lab.wporter.org'
sed -i "s/placeholder.domain.example/${domain_name}/g" /etc/nginx/conf.d/proxmox.conf
```

Now, let's run certbot to get your first certs.

First, create the acme-challenge web root directory:

```sh
mkdir -p /var/www/letsencrypt/.well-known/acme-challenge
chown -R www-data:www-data /var/www/letsencrypt
```

Then, tell certbot to ping your CA:

```sh
domain_name='800g4m.lab.wporter.org'
ca_acme_uri='https://intermediate-ca.lab.wporter.org/acme/acme/directory'
certbot certonly \
  --webroot -w /var/www/letsencrypt \
  --preferred-challenges http \
  --server "$ca_acme_uri" \
  --no-eff-email \
  --agree-tos \
  --non-interactive \
  -d "$domain_name"
```

Finally, assuming all is well, start NGINX:

```sh
systemctl enable --now nginx
```

Not quite done yet. More certbot jazz. RHEL & co provide an `/etc/sysconfig/certbot` file for env variables, which is super nice, and I like to put simple `restart NGINX` deploy hooks there.

Debian doesn't have that, so we'll need to write a very simple script and put it in the `/etc/letsencrypt/renewal-hooks/deploy` directory instead:

```sh
tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh > /dev/null << 'EOF'
#!/bin/bash
systemctl reload nginx.service
EOF
chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

This will make sure that NGINX can use the updated cert once Certbot has run. Since I run short-lived (24H) certificates, this is important for my environment!

Finally, good news! The Certbot timer does not need to be manually started on Debian, so that's all set. If you want to check on it, run a `systemctl list-timers certbot.timer` or `systemctl status certbot.timer`.

At this point, things should be working properly. Try to connect to PVE at fqdn.com:443 - if all has gone right, it'll work fine. Connecting via fqdn.com:80 should redirect you to the SSL-enabled site, too.

## PBS

You're going to want to use the system certificate manager so your Proxmox machines trust the PBS server's cert.

Again, in this case, your config is going to be pretty simple:

Install NGINX, and remove the default configuration:

```sh
apt install -y nginx
rm /etc/nginx/sites-enabled/default
```

Drop in a basic NGINX configuration to proxy 443 to localhost:8007 using PBS SSL certs:

```sh
tee /etc/nginx/conf.d/proxmox.conf > /dev/null << 'EOT'
upstream pbs {
 server "127.0.0.1:8007";
}

server {
 listen 443 ssl;
 listen [::]:443 ssl;
 ssl_certificate /etc/proxmox-backup/proxy.pem;
 ssl_certificate_key /etc/proxmox-backup/proxy.key;
 proxy_redirect off;
 location / {
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  proxy_pass https://pbs;
 }
}
EOT
```

Test the configuration:

```sh
nginx -t
```

If it passes, enable & start NGINX:

```sh
systemctl enable --now nginx
```

Test with cURL or your preferred flavor of Chromium.
