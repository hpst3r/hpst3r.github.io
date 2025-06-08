---
title: "Changing a Linux username"
date: 2025-06-07T20:30:00-00:00
draft: false
---

Super easy:

First, call usermod -l (change login) with the -d (home) and -m (move home) args.
This changes the username, moves the old home directory to the new home directory:

```sh
sudo usermod -l new_name -d /home/new_name -m old_name
```

Next, call groupmod --new-name to rename the user's group.

```sh
sudo groupmod --new-name new_group_name old_group_name
```

Annd.. you're done! See? Super easy.
