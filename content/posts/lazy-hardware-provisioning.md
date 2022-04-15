+++
author = "Alex Wicks"
title = "Lazy Hardware Provisioning"
date = "2022-04-15"
description = "Automated provisioning of hardware"
+++

Don't you ever find it annoying setting up your new laptop for the _nth time_? You always inevitably forget that one package which you need when you're in the middle of something, and it's never quite the same as the last install. Not to mention the time wasted - for me at least, it's normally a whole day set aside. Seeing as I like to automate things, this wasn't going to cut it.

None of what is mentioned here are novel concepts - solutions such as [Tinkerbell](https://tinkerbell.org/) have existed for a while. But who wants all that infrastructure for an ad-hoc build at home?

## Packer

Enter [Hashicorp Packer](https://www.packer.io/), a tool primarily designed for building VM templates.

But wait, you ask, how does this help me with my laptop? I don't want a VM!

## What's the difference anyway?

Although we might think of a VM as being a poor approximation of a real piece of hardware, long gone are the days of old with Windows driver hell. In fact, for the longest time people have been picking up their Linux install from one machine and copying it verbatim to another. So why not the same concept from a VM onto our bare metal? After all, the vast majority of drivers you'd ever need are in the kernel tree and included as modules by your distro in all likelihood.

## QEMU to the rescue

Packer has a great [QEMU builder](https://www.packer.io/plugins/builders/qemu), with a rich plugin system allowing us to add arbitrary steps before & after the image build. Any recent kernel and somewhat recent CPU will have the relevant hardware acceleration support in the form of KVM & Intel's VT-x or AMD's SVM respectively. QEMU itself is available on pretty much any distro, and can run inside a privileged Docker container in a pinch.

## How on earth does this work?

In a lot of cases where you're using Packer, you'll already have the luxury of a base image, such an AWS AMI that's already been provided, where you just want to add some customisations. In our case, we're not so lucky. And since I am using Arch Linux, the process is a little convoluted... [1]

### Preparing the environment

Just like doing it all by hand, we're booting into an Arch Linux live environment, where we are dropped into a shell. Except we want to run all the commands automatically. As such we need to setup the SSH communicator that Packer will be expecting. Luckily, the QEMU builder supports the `boot_command` directive, which allows us to enter a string which will then get typed on the keyboard.
```
boot_command = [
  "<enter><wait20s><enter>",
  "/usr/bin/curl -O http://{{ .HTTPIP }}:{{ .HTTPPort }}/prepare.sh",
  "<enter><wait2s>",
  "/usr/bin/curl -O http://{{ .HTTPIP }}:{{ .HTTPPort }}/inst_liveboot.sh",
  "<enter><wait2s>",
  "/usr/bin/curl -O http://{{ .HTTPIP }}:{{ .HTTPPort }}/dd.py",
  "<enter><wait2s>",
  "/usr/bin/curl -O http://{{ .HTTPIP }}:{{ .HTTPPort }}/seed.iso",
  "<enter><wait2s>",
  "/bin/sh ./prepare.sh",
  "<enter>"
]
```
#### Dissecting the boot command
```
"<enter><wait20s><enter>",
```
The values inside angle brackets are interpreted by Packer as escape sequences. At the beginning we are hitting enter to pass the bootloader screen, waiting for the live boot to finish booting for 20 seconds, then hitting enter again to make the auto-login worked fine and we have a clean shell ready for our commands.
```
"/usr/bin/curl -O http://{{ .HTTPIP }}:{{ .HTTPPort }}/prepare.sh",
```
Packer will handily serve a directory via its inbuilt HTTP server, if desired. Anything inside double curly braces is a template variable, and will be populated with the actual value by Packer. As such, I can place a simple script in my HTTP directory to configure SSH:
```
#!/bin/sh

set -e

echo 'root:init' | chpasswd -c SHA512
echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
systemctl restart sshd
```
And rinse and repeat for any other files I may need in my bootstrapping environment.
```
 "/bin/sh ./prepare.sh",
 "<enter>"
 ```
 Great - now we have all our files in place! Let's run our script so we can get the SSH communicator working.
 
#### Putting the SSH communicator to use

Packer has some neat provisioning plugins we can use, now that we have a channel of communication to the guest, such as the Ansible provisioner.
For our use case though, the shell provisioner should be sufficient. Note that it will copy the file onto the guest for you, no manual steps required.
```
provisioner "shell" {
  script = "packer/bootstrap_liveboot.sh"
}
 ```

From this point onwards, we just want to configure an Arch Linux system the way we want - partition the disk, make the filesystems, mount them under `/mnt`, run `pacstrap` to install the base system including any useful tools that might be needed (we'll touch on this later), generate the fstab file. All the usual stuff. With one twist...

#### Chroot environment

In a normal Arch Linux install, once you've installed & configured the base system, you would run `arch-chroot` which mounts the relevant bind mounts & chroots into the new system on your mount point. Except this doesn't work for us, since it won't allow us to run a script non-interactively. As such, we use a feature of systemd called [systemd-nspawn](https://www.freedesktop.org/software/systemd/man/systemd-nspawn.html) which is somewhat analagous to Docker, but using our filesystem on the disk as our 'image'.

First off, we need to copy any files needed _inside_ the new root:
```
for FILE in inst.sh postinst.sh
do
  cp "/root/${FILE}" /mnt
  chmod +x "/mnt/${FILE}"
done
```
Now we can fire off systemd-nspawn, with our 'entrypoint' being a script of our choosing which will be non-interactively executed inside our container environment, passing any relevant arguments you may want:
```
systemd-nspawn \
  -D /mnt \
  /inst.sh \
  "${CRYPT_PASSWD}" \
  "$(blkid -s UUID -o value "/dev/disk/by-partlabel/${ROOT_PARTLABEL}")"
```
Note how the path to the executable is now relative to the chrooted `/mnt` (i.e. if the file is at `/mnt/inst.sh`, since we chroot, that becomes `/inst.sh`).

Again, inside this script, you can configure your system however you like. Note that this environment will be a virtual machine, so don't rely on trying to detect what GPU is installed so you can install the right drivers, for example.

Once this finishes, you'll have a base image, very much like what an AWS AMI would be.

### OK, I have my image - now what?

What degree do you want to go to? The simplest thing here is to take the image we just created, and `dd` it directly onto your laptop's disk from a live boot.

But that's not good enough for me. Time for image-ception...

### Building a custom liveboot

To make life maximally easy, I build my own custom liveboot image, with the disk image we just created baked in. As soon as the livebooot starts, we run a python script which will prompt the user which disk they would like to write to, and then `dd` the disk image onto it. In addition, we will add some `cloud-init` configuration onto our liveboot USB stick, so that the machine can use this on first boot after copying the disk image across. This allows the machine to get its network configuration without me manually doing anything.

The process is much the same as before, using Packer to build another image which contains our first image and auto-launches the python script.

### cloud-init?

Cloud-init is another tool typically used to configure a cloud image at first boot. It talks to the cloud provider's metadata API to get some basic information about itself such as its hostname, as well as a user-provided file with any extra desired configuration (such as SSH keys). But we're not in a cloud environment - so what gives?

Well, cloud-init has a [NoCloud](https://cloudinit.readthedocs.io/en/latest/topics/datasources/nocloud.html) data source. Which handily means we can inject our configuration using the same USB stick we put our liveboot onto, by simply creating a partition with the correct label that cloud-init looks for, and putting our config there.

Why do we want this, you may ask?
 - It will grow the disk for us on first boot. Which is really handy, since we can then make the disk image only the size it needs to be, and grow it once copied onto the target disk automagically.
 - It will configure the network automatically. Which allows us to run other automatic scripts that require network connectivity.
 - We can trigger config management to run. In my case, I configure my whole system with Ansible, so it'll set up everything just how I like it, from installing all my packages down to even my dotfiles.

## Wrapping up

If you want to see an example, [here's how I've done it](https://github.com/aw1cks/sloth).
All the ansible configuration is available [here](https://github.com/archlinux-ansible/).

Why not use nix, you might ask? Good question. I approached this as more of a learning exercise, which happened to be quite useful. I'm not comfortable with nix nor do I see myself putting the time in to become proficient, so this works well for me.

All in all, I think this is a useful technique to increase reproducability, and it's quite useful for bootstrapping some infrastructure from zero quickly. It's not the best approach perhaps, but it only requires a USB stick and another PC.

### Addendum
[1] In the time since I devised this method, [archinstall](https://wiki.archlinux.org/title/Archinstall) was released, which would probably simplify this process.
