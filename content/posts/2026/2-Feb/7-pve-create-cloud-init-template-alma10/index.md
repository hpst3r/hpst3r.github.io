---
title: "Creating a cloud-init VM template for AlmaLinux 10 on Proxmox VE"
date: 2026-02-26T22:30:00-00:00
draft: false
---

Just a quick example.

```sh
vmid=9001
image_name="AlmaLinux-10-GenericCloud-latest.x86_64.qcow2"
storage="local-lvm"

wget https://repo.almalinux.org/almalinux/10/cloud/x86_64/images/AlmaLinux-10-GenericCloud-latest.x86_64.qcow2 -O $image_name

qemu-img resize $image_name 32G

qm create $vmid --name "almalinux-10-genericcloud" --ostype l26 \
  --memory 2048 \
	--agent 1 \
	--bios ovmf --machine q35 --efidisk0 "$storage":0,pre-enrolled-keys=0 \
	--cpu host --socket 1 --cores 2 \
	--net0 virtio,bridge=bridge0

qm set $vmid --sata0 local-lvm:cloudinit

qm importdisk $vmid $image_name $storage

qm set $vmid --scsihw virtio-scsi-single --scsi0 local-lvm:vm-${vmid}-disk-2,discard=on,cache=writeback,iothread=1,ssd=1

qm set $vmid --boot order=scsi0

qm template $vmid
```
