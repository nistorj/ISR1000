# ISR1000

## How to enable guest shell

I decided I'd give guest shell a try on my ISR1100 series router.  I'm using it for the use-case of collecting logs locally at the "branch" site from the ME (Mobility Express) setup from the built-in ISR-1100AC AP (AP-1815i model).  I will be collecting syslog and snmp traps to the guest shell setup and storing the data in a mariaDB.

The documentation found at `https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/prog/configuration/1612/b_1612_programmability_cg/guest_shell.html` has the general inforamtion on how to setup and deploy although it does refer to some older commands based on the output from the IOS parser.

### Prerequisites
Before we begin we need to make sure there are a couple items covered:
- Install IOS-XE 16.12.1a or later (Gibraltar)
- Flash space requirement: 1100MB free (1.1 GB)


### Steps to Enable Guest Shell
#### 1. Enable IOx
Guest Shell container is managed using IOx. IOx is Cisco's Application Hosting Infrastructure for Cisco IOS XE devices. IOx enables hosting of applications and services developed by Cisco, partners, and third-party developers in network edge devices, seamlessly across diverse and disparate hardware platforms.

Let's enable IOx....
```
c1100#show iox-service

IOx Infrastructure Summary:
---------------------------
IOx service (CAF)         : Not Running
IOx service (HA)          : Not Supported
IOx service (IOxman)      : Not Running
Libvirtd   1.3.4          : Running

c1100#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
c1100(config)#iox
c1100(config)#^Z
c1100#

Syslog:
Oct 14 22:16:15.434 EDT: %UICFGEXP-6-SERVER_NOTIFIED_START: R0/0: psd: Server iox has been notified to start
Oct 14 22:18:02.310 EDT: %IM-2-IOX_ENABLEMENT: R0/0: ioxman: IOX is ready.

<< wait a min or two >>

c1100#show iox-service

IOx Infrastructure Summary:
---------------------------
IOx service (CAF) 1.8.0.2 : Running
IOx service (HA)          : Not Supported
IOx service (IOxman)      : Running
Libvirtd   1.3.4          : Running

c1100#
```

We now see that the IOx Service (CAF and IOxman) have both started successfully.

#### 2. Create VirtualPortGroup interface
The documentation refers to configuring GuestShell in a single CLI command, however this command does not work on this platform anymore.  This was designed for Cisco IOS XE Everest 16.6.x or earlier, it's just not started until later on in the documentation.

We need to create the VirtualPortGroup interface and assign it an IP address.  In my environment it will also be sitting on the inside NAT so it can access external hosts.

```
c1100#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
c1100(config)#interface VirtualPortGroup0
c1100(config-if)# ip address 10.0.99.1 255.255.255.0
c1100(config-if)# ip nat inside
c1100(config-if)# no ip proxy-arp
```

**NOTE**: Make sure to add the subnet you choose to your NAT ACL if required.

#### 3. Configure AppHosting information.
The documentation refers to using some out-dated commands according to the IOS parser, eg:

```
c1100(config)#app-hosting appid guestshell
c1100(config-app-hosting)#vnic gateway1 virtualportgroup 0 guest-interface 0 ?
  guest-ipaddress  Guest IP address
  <cr>             <cr>

c1100(config-app-hosting)#vnic gateway1 virtualportgroup 0 guest-interface 0
c1100(config-app-hosting)#
```

With this example the syslog output throws the following:

`Oct 14 22:27:39.064 EDT: %PARSER-5-HIDDEN: Warning!!! ' vnic gateway1 virtualportgroup 0 guest-interface 0 ' is a hidden command. Use of this command is not recommended/supported and will be removed in future.`

What I have found to be the correct CLI commands are:
```
c1100(config)#app-hosting appid guestshell
c1100(config-app-hosting)# app-vnic gateway0 virtualportgroup 0 guest-interface 0
c1100(config-app-hosting-gateway0)#  guest-ipaddress 10.0.99.10 netmask 255.255.255.0
c1100(config-app-hosting-gateway0)# app-default-gateway 10.0.99.1 guest-interface 0
```

These do not throw any error and work when enabling guest-shell.

**Note**: If you plan to configure IPv6 you need to supply the `netmask` entry.  Most people would know this as the `prefix` in IPv6, eg:
```
c1100(config-app-hosting-gateway0)#guest-ipaddress ?
  A.B.C.D     Application/Guest IP Address
  X:X:X:X::X  Application/Guest IP Address
c1100(config-app-hosting-gateway0)#guest-ipaddress 2600:AABB:1001:2199::10 netmask ?
  A.B.C.D     Netmask
  X:X:X:X::X  Netmask
c1100(config-app-hosting-gateway0)#
c1100(config-app-hosting-gateway0)#guest-ipaddress 2600:AABB:1001:2199::10 netmask ffff:ffff:ffff:ffff:0000:0000:0000:0000
c1100(config-app-hosting-gateway0)# exit
c1100(config-app-hosting)# app-default-gateway 2600:AABB:1001:2199::1 guest-interface 0


cfg snipit:
app-hosting appid guestshell
 app-vnic gateway0 virtualportgroup 0 guest-interface 0
  guest-ipaddress 2600:AABB:1001:2199::10 netmask ffff:ffff:ffff:ffff::
 app-default-gateway 2600:AABB:1001:2199::1 guest-interface 0
end
```

**IMPORTANT**: The IPv6 information does not work passing over to Guest Shell.  One workaround is to enable IPv6 on the VirtualPortGroup0 and allow RA (router advertisement) messages to go out.  You can always set a static in the configuration files however it may be easier to allow stateless auto config to just do its job.


#### 4. Enable Guest Shell running
Once we have the required configuration in place we can enable Guest Shell.  If you do not have the appropriate amount of space you will see an error such as the following:

```
c1100#guestshell enable
Interface will be selected if configured in app-hosting
Please wait for completion
% Error: Not enough space in the disk, needed 1026107392, got 762314752

c1100#
```

Once you free up the disk space and try again the output should resemble:
```
c1100#guestshell enable
Interface will be selected if configured in app-hosting
Please wait for completion
guestshell installed successfully
Current state is: DEPLOYED
guestshell activated successfully
Current state is: ACTIVATED
guestshell started successfully
Current state is: RUNNING
Guestshell enabled successfully

c1100#
```

syslog: `Oct 14 22:33:12.483 EDT: %IM-6-IOX_INST_INFO: R0/0: ioxman: IOX SERVICE guestshell LOG: Guestshell is up at 09/14/2019 22:33:12`

```
c1100#show app-hosting list
App id                                   State
---------------------------------------------------------
guestshell                               RUNNING

c1100#
```

**Note**: You may receive some space alarms as this does take up a fair bit of space.

  `Oct 14 22:33:18.009 EDT: %PLATFORM-4-LOWSPACE:  bootflash : low space alarm assert`



#### 5. Access Guest Shell
The easiest way to get into Guest Shell is to execute bash to dive right in:
```
c1100#guestshell run bash
[guestshell@guestshell ~]$ id
uid=1000(guestshell) gid=1000(guestshell) groups=1000(guestshell),100000(network-admin),5(tty),10(wheel)
[guestshell@guestshell ~]$ sudo su
[root@guestshell guestshell]# id
uid=0(root) gid=0(root) groups=0(root),100000(network-admin)
[root@guestshell guestshell]#
```

As you can see there is no root password required if you use `sudo su`.  If you choose to just utilize `su` then you will be presented with the following:
```
[guestshell@guestshell ~]$ su
You are required to change your password immediately (root enforced)
New password:
[guestshell@guestshell ~]$
```

### Steps to update and manage Linux/apps under Guest Shell

#### Information about the system
The GuestShell Linux setup is by default CentOS 7.5.1804 (AltArch) when deployed on IOS-XE 16.12.1a.  You can find more information about this flavour at `CentOS AltArch SIG - AArch64 (http://wiki.centos.org/SpecialInterestGroup/AltArch/AArch64)`.

```
[guestshell@guestshell ~]$ uname -a
Linux guestshell 4.4.180-armada-17.10.1 #1 SMP Fri Jun 7 14:46:05 PDT 2019 aarch64 aarch64 aarch64 GNU/Linux
[guestshell@guestshell ~]$ cat /etc/redhat-release
CentOS Linux release 7.5.1804 (AltArch)
[guestshell@guestshell ~]$
```

There are four (4) processor cores enabled for Guest Shell:

```
[guestshell@guestshell ~]$ cat /proc/cpuinfo
processor	: 0 (1..3)
BogoMIPS	: 50.00
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd08
CPU revision	: 1
```

Memory:

```
[guestshell@guestshell ~]$ free
              total        used        free      shared  buff/cache   available
Mem:        4037112     1528996      878184      133216     1629932     2091544
Swap:             0           0           0
[guestshell@guestshell ~]$
```

The initial list of processes:
```
[guestshell@guestshell ~]$ ps axuww
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   7504  3172 ?        Ss   02:33   0:00 /sbin/init
root        15  0.0  0.0   9132  1608 ?        Ss   02:33   0:00 /usr/lib/systemd/systemd-journald
root        31  0.0  0.0  17676  4024 ?        Ss   02:33   0:00 /usr/sbin/sshd -D
dbus        36  0.0  0.0   8200  1948 ?        Ss   02:33   0:00 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
root        41  0.0  0.0 160856  2884 ?        Ssl  02:33   0:00 /usr/sbin/rsyslogd -n
root        42  0.0  0.0   4412  1548 ?        Ss   02:33   0:00 /usr/lib/systemd/systemd-logind
root        43  0.0  0.0   2184   772 ?        Ss   02:33   0:00 /sbin/agetty --noclear ttyS1 vt220
root        44  0.0  0.0   4340  1352 ?        Ss   02:33   0:00 /usr/sbin/crond -n
root        45  0.0  0.0   2184   708 pts/3    Ss+  02:33   0:00 /sbin/agetty --noclear --keep-baud pts/3 115200 38400 9600 vt220
root        46  0.0  0.0   2184   704 pts/0    Ss+  02:33   0:00 /sbin/agetty --noclear ttyS0 vt220
root        47  0.0  0.0   2184   708 pts/1    Ss+  02:33   0:00 /sbin/agetty --noclear --keep-baud pts/1 115200 38400 9600 vt220
root        48  0.0  0.0   2184   704 pts/2    Ss+  02:33   0:00 /sbin/agetty --noclear --keep-baud pts/2 115200 38400 9600 vt220
root        68  0.0  0.1  20208  5028 ?        Rs   02:36   0:00 sshd: guestshell@pts/4
guestsh+    69  0.0  0.0   3736  1792 pts/4    Ss   02:36   0:00 bash
guestsh+   111  0.0  0.0   7932  1588 pts/4    R+   02:52   0:00 ps axuww
[guestshell@guestshell ~]$
```


#### Adding DNS resolvers and setting the hostname
Let's first complete a couple steps to get the basics going.  First we will set the hostname on the system:

```
[root@guestshell guestshell]# cat /etc/hostname
guestshell
[root@guestshell guestshell]# echo "c1100gs" > /etc/hostname
[root@guestshell guestshell]#
[root@guestshell guestshell]# cat /etc/hostname
c1100gs
[root@guestshell guestshell]#
```

Next we set some DNS resolvers so that we can't access resources by hostnames.  Here we edit /etc/resolv.conf with `vi` for example and use something similar to:


```
domain home.local
nameserver 9.9.9.9
nameserver 8.8.8.8
options timeout:2 attempts:2 rotate
```

**NOTE**: This information is not saved if you disable and re-enable Guest Shell.  One option may be to edit `/etc/rc.local` and put in some auto-populating inforamtion there.

For more detailed info on options for this file you can checkout `https://ibm.co/2oWO3Mh`.

#### Updating to the latest revision of the OS.
Just like any other CentOS installation the simplest way to get up to date is to simply run a `yum upgrade` on the system.  The initial upgrade will update 95 Packages.

**NOTE**: You will consistently run into a failed package update with the following -- you can ignore this.

```
Failed:
  shadow-utils.aarch64 2:4.1.5.1-24.el7                         shadow-utils.aarch64 2:4.6-5.el7
```

After the `yum upgrade` is complete, reboot the system and you will find that you have moved to a later revision:

was: `CentOS Linux release 7.5.1804 (AltArch)`

now: `CentOS Linux release 7.7.1908 (AltArch)`


