# Qemu_setup.md

## Credit
tatumroaquin: [tatumroaquin Gist Github](https://gist.github.com/tatumroaquin)

## Check Virtualization Support

### Check CPU has VM capabilities
```bash
lscpu | grep -i Virtualization
```
* `VT-x` for Intel
* `AMD-Vi` for AMD

### Ensure that your kernel includes KVM modules
```bash
zgrep CONFIG_KVM /proc/config.gz
```

* `y` = Yes (always installed)
* `m` = Loadable module

## Install QEMU, libvirt, viewers, and tools
```bash
sudo pacman -S qemu-full qemu-img libvirt virt-install virt-manager virt-viewer \
edk2-ovmf swtpm guestfs-tools libosinfo

yay -S tuned
```
* `qemu-full` - user-space KVM emulator, manages communication between hosts and VMs
* `qemu-img` - provides create, convert, modify, and snapshot, offline disk images
* `libvirt` - an open-source API, daemon, and tool for managing platform virtualization
* `virt-install` - CLI tool to create guest VMs
* `virt-manager` - GUI tool to create and manage guest VMs
* `virt-viewer` -  GUI console to connect to running VMs
* `edk2-ovmf` - enables UEFI support for VMs
* `swtpm` - [TPM (Trusted Platform Module)](https://support.microsoft.com/en-us/topic/what-is-tpm-705f241d-025d-4470-80c5-4feeb24fa1ee) emulator for VMs
* `guestfs-tools` - provides a set of extended CLI tools for managing VMs
* `libosinfo` - a library for managing OS information for virtualization.
* `tuned` - system tuning service for linux allows us to optimise the hypervisor for speed.

## VirtIO Drivers for Windows Guests
Go to the [Fedora People](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/) repository and download `virtio-win.iso`.

Save it anywhere on disk, and attach it to a CD-ROM it when creating Windows VM.

The default location on Debian/RedHat based is `/usr/share/virtio-win/

## Enable the libvirt daemon
* Here is the [documentation](https://libvirt.org/daemons.html#architectural-options) detailing the difference between __monolithic__ and __modular__ daemons.
* Choose between option 1 and 2 and then do a `reboot`.

### Option 1: Enable the modular daemon.
```bash
for drv in qemu interface network nodedev nwfilter secret storage; do
    sudo systemctl enable virt${drv}d.service;
    sudo systemctl enable virt${drv}d{,-ro,-admin}.socket;
done
```
* loop through virtualization systemd services necessary for the libvirt modular daemon.

### Option 2: Enable the monolithic daemon.
```bash
sudo systemctl enable libvirtd.service
```

## Verify Host Virtualization
```bash
sudo virt-host-validate qemu
```
If you receive warnings, proceed to their respective sections. Re-run the above command to check your changes.

## Intel CPU IOMMU Support
> WARN (IOMMU appears to be disabled in the kernel. Add intel_iommu=on to kernel cmdline arguments)

### Enable IOMMU using GRUB
1. Open your GRUB config
```bash
sudo vim /etc/default/grub
```

2. Add the following kernel module entries 
```
# /etc/default/grub
GRUB_CMDLINE_LINUX="... intel_iommu=on iommu=pt"
```

3. Regenerate your `grub.cfg` file
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo reboot
```

## AMD SEV Support
For AMU CPUs with SEV feature, you might receive this warning:
> WARN (AMD Secure Encrypted Virtualization appears to be disabled in kernel. Add kvm_amd.sev=1 to the kernel cmdline arguments)

If you are in Intel you may ignore the warning below, as this only affects AMD CPUs.
> WARN (Unknown if this platform has Secure Guest support).

You may refer to this bug report:
* <https://bugzilla.redhat.com/show_bug.cgi?id=1850351#c5>

Or this libvirt documentation:
* <https://libvirt.org/kbase/launch_security_sev.html>

### Option 1: Enable AMD SEV using modprobe
```bash
echo "options kvm_amd sev=1" >> /etc/modprobe.d/amd-sev.conf
sudo reboot
```

### Option 2: Enable AMD SEV using GRUB
1. Open your GRUB config and add kernel modules
```bash
sudo vim /etc/default/grub
```
```
# /etc/default/grub
GRUB_CMDLINE_LINUX="... mem_encrypt=on kvm_amd.sev=1"
```

2. Regenerate your `grub.cfg` file
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo reboot
```

## Optimise Host with TuneD
1. Enable TuneD daemon
```bash
sudo systemctl enable --now tuned.service
```

2. Check active TuneD profile
```bash
tuned-adm active
```
> Current active profile: balanced
* `balanced` - generic profile not specialised for KVM, we will change this.

3. List all TuneD profiles
```bash
tuned-adm list
```

4. Set profile to `virtual-host`
```bash
sudo tuned-adm profile virtual-host
```

5. Verify that TuneD profile
```bash
tuned-adm active
```
> Current active profile: virtual-host
```bash
sudo tuned-adm verify
```
> Verification succeeded, current system settings match the preset profile.
See TuneD log file ('/var/log/tuned/tuned/log') for details.

## KVM Networking (optional)
* By default all virtual machines will connect to the built-in default NAT network.
* To make VMs accessible via the LAN you must create a network bridge.
* Keep in mind that network bridges won't work with hosts running on Wireless NICs.
* All configuration steps below is done using `NetworkManager`. If you use a different program to manage networking use that instead.

Default NAT network XML dump:
```xml
<network>
    <uuid>...</uuid>
    <forward mode='nat'>
        <nat>
            <port start='1024' end='65535'/>
        </nat>
    </forward>
    <bridge name='virbr0' stp='on' delay='0'/>
    <mac address='AB:CD:EF:AB:CD:EF'/>
    <ip address='10.1.1.1' netmask='255.255.255.0'>
        <dhcp>
            <range start="10.1.1.2" end="10.1.1.254"/>
        </dhcp>
    </ip>
</network>
```

### Sample Netfilter (nftables) rules for default NAT
The rules below is my personal config for my laptop machine.
Which works nicely with the autogenerated libvirt network rules.
```nftables
#!/usr/bin/nft -f

flush ruleset;

define qemu_iface = "virbr0";

table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;

		ct state established,related accept;

		iifname "lo" accept comment "allow loopback";
		iifname $qemu_iface accept comment "allow qemu";

		tcp dport http accept comment "allow sending http";
		tcp dport https accept comment "allow sending https";
		udp dport 67 udp sport 68 accept comment "allow sending dhcp";
		tcp dport ssh accept comment "allow ssh";

		counter drop;
	}

	chain forward {
		type filter hook forward priority filter; policy drop;

		ct state established,related accept;

		iifname $qemu_iface accept comment "forward qemu input";
		oifname $qemu_iface accept comment "forward qemu output";

		counter drop;
	}
}

table ip nat {
	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		ip saddr 10.1.1.0/24 masquerade;
	}
}
```

### Configure bridge interface

1. find the interface name of your ethernet connection.
```bash
sudo nmcli device status
```

| DEVICE | TYPE     |  STATE                 | CONNECTION         |
|:------:|:--------:|:----------------------:|:------------------:|
| enp2s0 | ethernet | connected              | Wired connection 1 |
| lo     | loopback | connected (externally) | lo                 |
| virbr0 | bridge   | connected (externally) | virbr0             |

2. create a bridge interface using `nmcli`
```bash
sudo nmcli connection add type bridge con-name bridge0 ifname bridge0
```

3. connect the ethernet interface to the bridge
```bash
sudo nmcli connection add type ethernet slave-type bridge con-name 'Bridge connection 1' \
ifname enp2s0 master bridge0
```

4. active the newly created connection
```bash
sudo nmcli connection up bridge 0
```

5. enable `connection.autoconnect-slaves` parameter.
```bash
sudo nmcli connection modify bridge0 connection.autoconnect-slaves 1
```

6. reactivate the bridge and verify connection.
```bash
sudo nmcli connection up bridge0
sudo nmcli device status
```

| DEVICE  | TYPE     |  STATE                 | CONNECTION          |
|:-------:|:--------:|:----------------------:|:-------------------:|
| bridge0 | bridge   | connected              | bridge0             |
| lo      | loopback | connected (externally) | lo                  |
| virbr0  | bridge   | connected (externally) | virbr0              |
| enp2s0  | ethernet | connected              | Bridge connection 1 |

### Configure bridge network
1. create an XML file called `nwbridge.xml`.
```bash
vim nwbridge.xml
```

2. post the following XML
```xml
<network>
    <name>nwbridge</name>
    <forward mode="bridge" />
    <bridge name="bridge0" />
</network>
```

3. define the bridge network
```bash
sudo virsh net-define nwbridge.xml
```
> Network nwbridge defined from nwbridge.xml

4. start the bridge network
```bash
sudo virsh net-start nwbridge
```

5. auto-start bridge network on boot
```bash
sudo virsh net-autostart nwbridge
```

6. delete `nwbridge.xml` file
```bash
rm nwbridge.xml
```

7. verify that `nwbridge` network exists.
```bash
sudo virsh net-list --all
```
| Name | State | Autostart | Persistent |
|:----:|:-----:|:---------:|:----------:|
| default | active | yes | yes |
| nwbridge | active | yes | yes |

### Removing bridge network and interface
If you want to revert the changes to your network, do the following:
```bash
sudo virsh net-destroy nwbridge
sudo virsh net-undefine nwbridge
sudo nmcli connection up 'Wired connection 1'
sudo nmcli connection del bridge0
sudo nmcli connection del 'Bridge connection 1'
```

## Libvirt connection modes
libvirt as two methods for connecting to the KVM Hypervisor.

### Session Mode
In `session` mode, a regular user is connected to a per-user instance. Allowing each user to manage their own pool of virtual machines. This is also the default mode.

The advantage of this mode is, permissions are not an issue. As no root access is required.

The disadvantage is this mode uses [QEMU User Networking (SLIRP)](https://wiki.qemu.org/Documentation/Networking#User_Networking_(SLIRP)). This is a user-space IP stack, which yields overhead resulting in poor networking performance.

And if you want to implement an option that requires `root` privileges. You will be unable to do so.

### System Mode
In the `system` mode you are granted access to all system resources.

### Granting system-wide access to regular user.
1. check current mode
```bash
virsh uri
```
> qemu:///session
```bash
sudo virsh uri
```
> qemu:///system
- This what we want

2. add the current user to the `libvirt` group
```bash
sudo usermod -aG libvirt $USER
```

3. set env variable with the default uri and check
```bash
echo 'export LIBVIRT_DEFAULT_URI="qemu:///system"' >> ~/.bashrc
```
```bash
source ~/.bash.rc
```
```bash
virsh uri
```
> should be qemu:///system

## Set ACL for the KVM images directory

1. check permissions on the images directory
```bash
sudo getfacl /var/lib/libvirt/images
```
```bash
getfacl: Removing leading '/' from absolute path names
# file : var/lib/libvirt/images/
# owner: root
# group: root
user::rwx
group::--x
other::--x
```

2. recursively remove existing ACL permissions
```bash
sudo setfacl -R -b /var/lib/libvirt/images/
```

3. recursively grant permission to the current user
```bash
sudo setfacl -R -m u:${USER}:rwX /var/lib/libvirt/images/
```
* uppercase `X` states that execution permission only applied to child folders and not child files.

4. enable special permissions default ACL
```bash
sudo setfacl -m d:u:${USER}:rwx /var/lib/libvirt/images/
```
* if this step is omitted, new dirs or files created within the images directory will not have this ACL set.

5. verify your ACL permissions within the images directory.
```bash
sudo getfacl /var/lib/libvirt/images/
```
```bash
getfacl: Removing leading '/' from absolute path names
# file : var/lib/libvirt/images/
# owner: root
# group: root
user::rwx
user:tatum:rwx
group::--x
mask::rwx
other::--x
default:user::rwx
default:user:tatum:rwx
default:group::--x
default:mask::rwx
default:other::--x
```

