---
title: "Snippet: unlock a SSH key for use with Ansible"
date: 2024-12-15T12:34:56-00:00
draft: false
---

Pulled out of a readme.md, since it's easier for me to keep it in one place (here.)

Unlock a protected SSH key for use by Ansible:

```txt
liam@liam-p1g4i-0:~/projects/ansible-a9-hypervisor$ eval "$(ssh-agent -s)"
Agent pid 5695
liam@liam-p1g4i-0:~/projects/ansible-a9-hypervisor$ ssh-add
Enter passphrase for /home/liam/.ssh/id_ed25519:
Identity added: /home/liam/.ssh/id_ed25519 (/home/liam/.ssh/id_ed25519)

ansible-playbook --inventory inventory.ini configure.yaml
```