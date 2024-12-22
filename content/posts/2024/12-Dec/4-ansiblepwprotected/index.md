---
title: "Snippet: unlock a SSH key for use with Ansible"
date: 2024-12-15T12:34:56-00:00
draft: false
---

Pulled out of a readme.md, since it's easier for me to keep it in one place (here.)

To unlock a protected SSH key for use by Ansible:

```txt
$ eval "$(ssh-agent -s)"
Agent pid 5695
$ ssh-add
Enter passphrase for /home/user/.ssh/id_ed25519:
Identity added: /home/user/.ssh/id_ed25519 (/home/user/.ssh/id_ed25519)

ansible-playbook -i host configure.yaml
```