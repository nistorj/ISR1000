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

