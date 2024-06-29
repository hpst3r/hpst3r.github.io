---
title: "Setting up Vagrant with libvirt (Fedora 40)"
date: 2024-06-05T12:34:56-00:00
draft: false
---

*Hi! This is personal documentation (also known as 'my notes.') Quality is.. not guaranteed.*
WIP as of 6/10/24. I keep getting distracted by PowerShell and SQL

Adapted on 6/5/24 from:
- [Fedora Magazine article on Vagrant + libvirt](https://fedoramagazine.org/vagrant-qemukvm-fedora-devops-sysadmin/)
- [Jeff Geerling's Ansible for Devops book, pgs.~5-10](https://leanpub.com/ansible-for-devops/c/CTVMPCbEeXd3)
- [The Vagrant docs](https://developer.hashicorp.com/vagrant/docs)


## Quickstart

#### Install packages
```txt
# dnf install qemu-kvm libvirt libguestfs-tools virt-install rsync vagrant
# systemctl enable libvirtd && sudo systemctl start libvirtd
# vagrant plugin install vagrant-libvirt
```

#### Download a box
```txt
$ vagrant box add fedora/40-cloud-base --provider=libvirt
```

*Hey! Are you from the future? Is Fedora 40 old now?*

- [Find a list of Fedora Vagrant images here!](https://app.vagrantup.com/fedora) 
- [Find other Vagrant images (boxes) here!](https://app.vagrantup.com/boxes/search)

#### Write a Vagrantfile

```txt
$ mkdir vagrant-demo && cd vagrant-demo
[user:~/vagrant-demo]$ touch Vagrantfile
```
```txt
Vagrant.configure("2") do |config|
  
  # image to use
  config.vm.box = "fedora/40-cloud-base"

  config.vm.provider :libvirt do |libvirt|

    libvirt.cpus = 2
    libvirt.memory = 2048

    # the default qemu session connection cannot create networks
    # either disable it, like so:
    libvirt.qemu_use_session = false
    # or specify the connection uri:
    # libvirt.uri = 'qemu:///system'

  end

  config.vm.hostname = "localhost"

  # Mounting directories
  # (this is the implicit default, manually disabled)
  config.vm.synced_folder "./", "/vagrant", disabled: true

  # Networking
  # VAGRANT GUESTS ARE INSECURE BY DEFAULT
  # DO NOT EXPOSE THEM TO UNSECURED NETWORKS

  # config.vm.network "forwarded_port", guest: 443, host: 4433
  #
  # config.vm.network "public_network", ip: "192.168.1.1", hostname: true
  # hostname: true adds the hostname and interface IP to /etc/hosts
  #
  # config.vm.network "public_network"
  # no IP specified? DHCP will be used.
  #
  # By default, public networks will bridge to the
  # network interface on the host machine. If more
  # than one exists, Vagrant will ask which to use.
  # Or specify the interface like so:
  #
  # config.vm.network "public_network", bridge: "eno1"

end
```

Define multiple VMs in a Vagrantfile

```txt
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
    config.vm.box = "generic/rocky9"
    config.ssh.insert_key = false
    config.vm.synced_folder ".", "/vagrant", disabled: true
    config.vm.provider :libvirt do |lv|
        lv.cpus = 1
        lv.memory = 1024
        lv.qemu_use_session = false
    end

    config.vm.define "app1" do |app|
        app.vm.hostname = "app1.test"
        app.vm.network :private_network, ip: "192.168.99.10"
    end

    config.vm.define "app2" do |app|
        app.vm.hostname = "app2.test"
        app.vm.network :private_network, ip: "192.168.99.11"
    end

    config.vm.define "db" do |db|
        db.vm.hostname = "db1.test"
        db.vm.network :private_network, ip: "192.168.99.15"
    end
end
```

Worth noting that Fedora cloud guest images do not take assigned IPs. [Known issue, open since Fedora 36, as of 6/8/24](https://github.com/hashicorp/vagrant/issues/12762)

#### Start a VM with Vagrant

```txt
[root:~/vagrant-demo]# vagrant up
```

#### Connect to a Vagrant-managed VM (default)

```txt
[root:~/vagrant-demo]# vagrant ssh
```

## Further Configuration & Behavior

#### Make files accessible to the VM

[Vagrant docs](https://developer.hashicorp.com/vagrant/docs/synced-folders/basic_usage)

By default, Vagrant syncs the directory containing the Vagrantfile on the host with the `/vagrant` directory in the guest.

Below is the implicit default behavior explicitly specified:
```txt
  # "source" on host, "destination" on guest
  config.vm.synced_folder ".", "/vagrant"
```

Other quick examples:
```txt
  config.vm.synced_folder "data/", "/mnt/data"
```
```txt
  config.vm.synced_folder "src/", "/srv/website"
```

#### Configure networking

[Vagrant docs](https://developer.hashicorp.com/vagrant/docs/networking)

**Vagrant is intentionally NOT secure by default. EXERCISE CAUTION.**

Vagrant assumes that the first network device is NAT, so the host (and Vagrant) can always talk to the guest directly.

**Private networks** *should never* allow the general public to access your guest

**Public networks** *can* allow the general public to access your guest

