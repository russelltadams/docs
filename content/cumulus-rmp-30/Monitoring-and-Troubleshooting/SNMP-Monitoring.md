---
title: SNMP Monitoring
author: Cumulus Networks
weight: 115
aliases:
 - /display/RMP30/SNMP+Monitoring
 - /pages/viewpage.action?pageId=5118703
pageID: 5118703
---
Cumulus RMP utilizes the open source Net-SNMP agent `snmpd`, v5.7.3,
which provides support for most of the common industry-wide MIBs,
including interface counters and TCP/UDP IP stack data.

{{%notice note%}}

Cumulus RMP does not prevent customers from extending SNMP features.
However, Cumulus Networks encourages the use of higher performance
monitoring environments, rather than SNMP.

{{%/notice%}}

## Introduction to SNMP (Simple Network Management Protocol)</span>

SNMP is an IETF standards-based network management architecture and
protocol that traces its roots back to Carnegie-Mellon University in
1982. Since then, it's been modified by programmers at the University of
California. In 1995, this code was also made publicly available as the
UCD project. After that, `ucd-snmp` was extended by work done at the
University of Liverpool as well as later in Denmark. In late 2000, the
project name changed to net-snmp and became a fully-fledged
collaborative open source project. The version used by Cumulus Networks
is base on the latest `net-snmp` 5.7.3 branch with added custom MIBs and
pass through and pass persist scripts.

## Configuring Ports for SNMP to Listen for Requests</span>

For security reasons, the default port binding for `snmpd` is the
loopback local address; consequently by default, the SNMP service does
not listen for SNMP requests from outside the switch. In order to listen
to requests from outside the switch, you need to change this binding to
a specific IP address (or all interfaces) after configuring security
access (community strings, users, and so forth). This is a change from
older versions of Cumulus RMP (before version 3.0), which listened to
incoming requests on all interfaces by default. The `snmpd`
configuration file is `/etc/snmp/snmpd.conf` and should be modified
before enabling and starting `snmpd`. The default configuration has no
access community strings defined so `snmpd` will not respond to any SNMP
requests until this is added.

## Starting the SNMP Daemon</span>

The following procedure is the recommended process to start `snmpd` and
monitor it using `systemctl`.

To start the SNMP daemon:

1.  Start the `snmpd` daemon:
    
        cumulus@switch:~$ sudo systemctl start snmpd.service

2.  Configure the `snmpd` daemon to start automatically after reboot:
    
        cumulus@switch:~$ sudo systemctl enable snmpd.service

3.  To enable `snmpd` to restart automatically after failure:
    
    1.  Create a file called
        `/etc/systemd/system/snmpd.service.d/restart.conf`.
    
    2.  Add the following lines:
        
            [Service]
            Restart=always
            RestartSec=60
    
    3.  Run `systemctl daemon-reload`.

Once the service is started, SNMP can be used to manage various
components on the Cumulus RMP switch.

## Configuring SNMP</span>

Cumulus RMP ships with a production usable default `snmpd.conf` file
included. This section covers a few basic configuration options in
`snmpd.conf`. For more information regarding further configuring this
file, refer to the `snmpd.conf` man page.

{{%notice warning%}}

The default `snmpd.conf` file does not include all supported MIBs or
OIDs that can be exposed.

{{%/notice%}}

{{%notice note%}}

Customers must at least change the default community string for v1 or
v2c environments or the `snmpd` daemon will not respond to any requests.

{{%/notice%}}

### Setting up the Custom Cumulus Networks MIBs</span>

{{%notice note%}}

No changes are required in the `/etc/snmp/snmpd.conf` file on the
switch, in order to support the custom Cumulus Networks MIBs. The
following lines are already included by default:

    view systemonly included .1.3.6.1.4.1.40310.1
    view systemonly included .1.3.6.1.4.1.40310.2
    sysObjectID 1.3.6.1.4.1.40310
    pass_persist .1.3.6.1.4.1.40310.1 /usr/share/snmp/resq_pp.py
    pass_persist .1.3.6.1.4.1.40310.2 /usr/share/snmp/cl_drop_cntrs_pp.py

{{%/notice%}}

However, several files need to be copied to the server, in order for the
custom Cumulus MIB to be recognized on the destination NMS server.

  - `/usr/share/snmp/Cumulus-Snmp-MIB.txt`

  - `/usr/share/snmp/Cumulus-Counters-MIB.txt`

  - `/usr/share/snmp/Cumulus-Resource-Query-MIB.txt`

### Enabling the .1.3.6.1.2.1 Range</span>

Some MIBs, including storage information, are not included by default in
`snmpd.conf` in Cumulus RMP. This results in some default views on
common network tools (like `librenms`) to return less than optimal data.
More MIBs can be included, by enabling all the .1.3.6.1.2.1 range. This
simplifies the configuration file, removing concern that any required
MIBs will be missed by the monitoring system. Various new MIBs were
added for 3.0 and include the following: ENTITY and ENTITY-SENSOR MIB
and parts of the BRIDGE-MIB and Q-BRIDGE-MIBs. These are included in the
default configuration (Note: the view of the BRIDGE-MIB and Q-BRIDGE-MIB
are commented out).

{{%notice warning%}}

This configuration grants access to a large number of MIBs, including
all MIB2 MIBs, which could reveal more data than expected. In addition
to being a security vulnerability, it could consume more CPU resources.

{{%/notice%}}

To enable the .1.3.6.1.2.1 range:

1.  Open `/etc/snmp/snmpd.conf` in a text editor.

2.  Make sure the following lines are included in the configuration:
    
        ###############################################################################
        #
        #  ACCESS CONTROL 
        #
        
        # system
        view   systemonly  included   .1.3.6.1.2.1
        # lldpd (Note: lldpd must be restarted with the -x option
    
        #     configured in order to send info to snmpd via Agent X
    
        view   systemonly  included   .1.0.8802.1.1.2
        # Cumulus specific
        view   systemonly  included   .1.3.6.1.4.1.40310.1
        view   systemonly  included   .1.3.6.1.4.1.40310.2

3.  Restart `snmpd`:
    
        # sudo systemctl start snmpd.service

### Enabling Public Community</span>

The `snmpd` authentication for versions 1 and 2 is disabled by default
in Cumulus RMP. This password (called a community string) can be enabled
by setting **rocommunity** (for read-only access) or **rwcommunity**
(for read-write acces). To enable read-only querying by a client:

1.  Open `/etc/snmp/snmpd.conf` in a text editor.

2.  To allow read-only access using a password *public* from any client
    IP address (*default*) for the view you defined before with
    *systemonly*, add the following line to the end of the file, then
    save it:
    
        rocommunity public default -V systemonly

3.  Restart `snmpd`:
    
        cumulus@switch:~$ sudo systemctl restart snmpd.service

### Configuring SNMPv3</span>

Since community strings in versions 1 and 2c are sent in the clear,
SNMPv3 is often used to enable authentication and encryption. SNMPv3 was
first release around 2000. A minimal example is shown here for
`/etc/snmp/snmpd.conf` that defines three users, each with a different
combination of authentication and encryption. Please change these
usernames and passwords before using this in a network:

{{%notice note%}}

Make sure you change the usernames and passwords in the sample code
below, as the ones used here are for explanatory purposes only.

{{%/notice%}}

    # simple no auth user
    createUser user1
    # user with MD5 authentication
    createUser user2 MD5 user2password
    # user with MD5 for auth and DES for encryption 
    createUser user3 MD5 user3password DES user3encryption
    # user999 with MD5 for authentication and DES for encryption
      
    createUser user666 SHA user666password AES user666encryption
    createUser user999 MD5 user999password DES user999encryption
    
    # restrict users to certain OIDs
    # (Note: creating rouser or rwuser will give
    # access regardless of the createUser command above. However,
    # createUser without rouser or rwuser will not provide any access).
    rouser user1 noauth 1.3.6.1.2.1.1
    rouser user2 auth 1.3.6.1.2.1
    rwuser user3 priv 1.3.6.1.2.1
    rwuser user666
    rwuser user999

Once you make this configuration and restart the `snmpd` daemon, the
user access can be checked with a client - the Debian package called
`snmp` contains `snmpget` and `snmpwalk`, as well as other programs that
are useful for checking daemon functionality from the switch itself or
from another workstation. The following commands check the access for
each user defined above from the localhost (the switch itself):

    # check user1 which has no authentication or encryption (NoauthNoPriv)
    snmpget -v 3 -u user1 -l NoauthNoPriv localhost 1.3.6.1.2.1.1.1.0
    snmpwalk -v 3 -u user1 -l NoauthNoPriv localhost 1.3.6.1.2.1.1
    
    # check user2 which has authentication but no encryption (authNoPriv)
    snmpget -v 3 -u user2 -l authNoPriv -a MD5 -A user2password localhost 1.3.6.1.2.1.1.1.0
    snmpget -v 3 -u user2 -l authNoPriv -a MD5 -A user2password localhost 1.3.6.1.2.1.2.1.0
    snmpwalk -v 3 -u user2 -l authNoPriv -a MD5 -A user2password localhost 1.3.6.1.2.1
      
    # check user3 which has both authentication and encryption (authPriv)
    snmpget -v 3 -u user3 -l authPriv -a MD5 -A user3password -x DES -X user3encryption localhost .1.3.6.1.2.1.1.1.0
    snmpwalk -v 3 -u user3 -l authPriv -a MD5 -A user3password -x DES -X user3encryption localhost .1.3.6.1.2.1

    snmpwalk -v 3 -u user666 -l authPriv -a SHA -x AES -A user666password -X user666encryption localhost 1.3.6.1.2.1.1
    snmpwalk -v 3 -u user999 -l authPriv -a MD5 -x DES -A user999password -X user999encryption localhost 1.3.6.1.2.1.1

A slightly more secure method of configuring SNMPv3 users without
creating cleartext passwords is the following:

1.  Install the `net-snmp-config` script that is in `libsnmp-dev`
    package:
    
        cumulus@switch:~$ sudo apt-get update
        cumulus@switch:~$ sudo apt-get install libsnmp-dev

2.  Stop the daemon:
    
        cumulus@switch:~$ sudo systemctl stop snmpd

3.  Use the `net-snmp-config` command to create two users, one with MD5
    and DES, and the next with SHA and AES.
    
    {{%notice note%}}
    
    The minimum password length is 8 characters and the arguments `-a`
    and `-x` to `net-snmp-config` have different meanings than they do
    for `snmpwalk`.
    
    {{%/notice%}}
    
        cumulus@switch:~$ sudo net-snmp-config --create-snmpv3-user -a md5authpass -x desprivpass -A MD5 -X DES userMD5withDES 
        cumulus@switch:~$ sudo net-snmp-config --create-snmpv3-user -a shaauthpass -x aesprivpass -A SHA -X AES userSHAwithAES
        cumulus@switch:~$ sudo systemctl start snmpd

This adds a `createUser` command in `/var/lib/snmp/snmpd.conf`. Do
**not** edit this file by hand, unless you are removing usernames. It
also adds the rwuser in `/usr/share/snmp/snmpd.conf`. You may want to
edit this file and restrict access to certain parts of the MIB by adding
*noauth*, *auth* or *priv* to allow unauthenticated access, require
authentication or to enforce use of encryption, respectively.

The `snmpd` daemon reads the information from the
`/var/llib/snmp/snpmd.conf` file and then the line is removed
(eliminating the storage of the master password for that user) and
replaced with the key that is derived from it (using the EngineID). This
key is a localized key, so that if it is stolen it cannot be used to
access other agents. To remove the two users userMD5withDES and
userSHAwithAES, you need simply stop the `snmpd` daemon and edit the
files `/var/lib/snmp/snmpd.conf` and `/usr/share/snmp/snmpd.conf`.
Simply remove the lines containing the username. Then restart the
`snmpd` daemon as in step 3 above.

From a client, you would access the MIB with the correct credentials.
(Again, note that the roles of `-x`, `-a` and `-X` and `-A` are reversed
on the client side as compared with the `net-snmp-config` command used
above.)

    snmpwalk -v 3 -u userMD5withDES -l authPriv -a MD5 -x DES -A md5authpass -X desprivpass localhost 1.3.6.1.2.1.1.1
    snmpwalk -v 3 -u userSHAwithAES -l authPriv -a SHA -x AES -A shaauthpass -X aesprivpass localhost 1.3.6.1.2.1.1.1

## Configuring Nutanix Prism</span>

Nutanix Prism is a graphical user interface (GUI) for managing
infrastructures and virtual environments.

### Cumulus RMP Configuration</span>

1.  SSH to the Cumulus RMP switch that needs to be configured, replacing
    `[switch]` below as appropriate:
    
        cumulus@workbench:~$ ssh cumulus@[switch]

2.  Confirm the switch is running Cumulus RMP 2.5.5 or newer:
    
        cumulus@switch$ cat /etc/lsb-release
        DISTRIB_ID="Cumulus RMP"
        DISTRIB_RELEASE=2.5.5
        DISTRIB_DESCRIPTION=2.5.5-4cd66d9-201512071809-build

3.  Open the `/etc/snmp/snmpd.conf` file in an editor.

4.  Uncomment the following 3 lines in the `/etc/snmp/snmpd.conf` file,
    and save the file:
    
      - bridge\_pp.py
        
            pass_persist .1.3.6.1.2.1.17 /usr/share/snmp/bridge_pp.py
    
      - Community
        
            rocommunity public  default    -V systemonly
    
      - Line directly below the Q-BRIDGE-MIB (.1.3.6.1.2.1.17)
        
            # BRIDGE-MIB and Q-BRIDGE-MIB tables
            view   systemonly  included   .1.3.6.1.2.1.17

5.  Restart `snmpd`:
    
        cumulus@switch$ sudo systemctl restart snmpd.service
    
        Restarting network management services: snmpd.
        cumulus@switch$

### Nutanix Configuration</span>

1.  Log into the Nutanix Prism. Nutanix defaults to the Home menu,
    referred to as the Dashboard:
    
    {{% imgOld 0 %}}

2.  Click on the gear icon
    
    {{% imgOld 1 %}}
    
    in the top right corner of the dashboard, and select NetworkSwitch:
    
    {{% imgOld 2 %}}

3.  Click the **+Add Switch Configuration** button in the **Network
    Switch Configuration** pop up window.

4.  Fill out the **Network Switch Configuration** for the Top of Rack
    (ToR) switch configured for snmpd in the previous section:
    
    {{% imgOld 3 %}}
    
    | Configuration Parameter         | Description                                                                                     | Value Used in Example                                   |
    | ------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
    | Switch Management IP Address    | This can be any IP address on the box. In the screenshot above, the eth0 management IP is used. | 192.168.0.111                                           |
    | Host IP Addresses or Host Names | IP addresses of Nutanix hosts connected to that particular ToR switch.                          | 192.168.0.171,192.168.0.172,192.168.0.173,192.168.0.174 |
    | SNMP Profile                    | Saved profiles, for easy configuration when hooking up to multiple switches.                    | None                                                    |
    | SNMP Version                    | SNMP v2c or SNMP v3. Cumulus RMP has only been tested with SNMP v2c for Nutanix integration.    | SNMP v2c                                                |
    | SNMP Community Name             | SNMP v2c uses communities to share MIBs. The default community for snmpd is 'public'.           | public                                                  |
    

    {{%notice note%}}
    
    The rest of the values were not touched for this demonstration. They
    are usually used with SNMP v3.
    
    {{%/notice%}}

5.  Save the configuration. The switch will now be present in the
    **Network Switch Configuration** menu now.

6.  Close the pop up window to return to the dashboard.

7.  Open the **Hardware** option from the **Home** dropdown menu:
    
    {{% imgOld 4 %}}

8.  Click the **Table** button.

9.  Click the **Switch** button. Configured switches are shown in the
    table, as indicated in the screenshot below, and can be selected in
    order to view interface statistics:
    
    {{% imgOld 5 %}}

{{%notice note%}}

The switch has been added correctly, when interfaces hooked up to the
Nutanix hosts are visible.

{{%/notice%}}

## Switch Information Displayed on Nutanix Prism</span>

  - Physical Interface (e.g. swp1, swp2). This will only display swp
    interfaces connected to Nutanix hosts by default.

  - Switch ID - Unique identifier that Nutanix keeps track of each port
    ID (see below)

  - Index - interface index, in the above demonstration swp49 maps to
    Index 52 because there is a loopback and two ethernet interface
    before the swp starts.

  - MTU of interface

  - MAC Address of Interface

  - Unicast RX Packets (Received)

  - Unicast TX Packets (Transmitted)

  - Error RX Packets (Received)

  - Error TX Packets (Transmitted)

  - Discard RX Packets (Received)

  - Discard TX Packets (Transmitted)

The Nutanix appliance will use Switch IDs that can also be viewed on the
Prism CLI (by SSHing to the box). To view information from the Nutanix
CLI, login using the default username **nutanix**, and the password
**nutanix/4u**.

    nutanix@NTNX-14SM15270093-D-CVM:192.168.0.184:~$ ncli network list-switch
        Switch ID                 : 00051a76-f711-89b6-0000-000000003bac::5f13678e-6ffd-4b33-912f-f1aa6e8da982
        Name                      : switch
        Switch Management Address : 192.168.0.111
        Description               : Linux switch 3.2.65-1+deb7u2+cl2.5+2 #3.2.65-1+deb7u2+cl2.5+2 SMP Mon Jun 1 18:26:59 PDT 2015 x86_64
        Object ID                 : enterprises.40310
        Contact Information       : Admin <admin@company.com>
        Location Information      : Raleigh, NC
        Services                  : 72
        Switch Vendor Name        : Unknown
        Port Ids                  : 00051a76-f711-89b6-0000-000000003bac::5f13678e-6ffd-4b33-912f-f1aa6e8da982:52, 00051a76-f711-89b6-0000-000000003bac::5f13678e-6ffd-4b33-912f-f1aa6e8da982:53, 00051a76-f711-89b6-0000-000000003bac::5f13678e-6ffd-4b33-912f-f1aa6e8da982:54, 00051a76-f711-89b6-0000-000000003bac::5f13678e-6ffd-4b33-912f-f1aa6e8da982:55

## Troubleshooting</span>

To help visualize the following diagram is provided:

{{% imgOld 6 %}}

| Nutanix Node    | Physical Port | Cumulus RMP Port |
| --------------- | ------------- | ---------------- |
| Node A (Green)  | vmnic2        | swp49            |
| Node B (Blue)   | vmnic2        | swp50            |
| Node C (Red)    | vmnic2        | swp51            |
| Node D (Yellow) | vmnic2        | swp52            |

## Enabling LLDP/CDP on VMware ESXi (Hypervisor on Nutanix)</span>

1.  Follow the directions on one of the following websites to enable
    CDP:
    
    1.  <http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1003885>
    
    2.  <http://wahlnetwork.com/2012/07/17/utilizing-cdp-and-lldp-with-vsphere-networking/>
        
        e.g. Switch CDP on:
        
            root@NX-1050-A:~] esxcli network vswitch standard set -c both -v vSwitch0
        
        Then confirm it is running:
        
            root@NX-1050-A:~] esxcli network vswitch standard list -v vSwitch0
            vSwitch0
               Name: vSwitch0
               Class: etherswitch
               Num Ports: 4082
               Used Ports: 12
               Configured Ports: 128
               MTU: 1500
               CDP Status: both
               Beacon Enabled: false
               Beacon Interval: 1
               Beacon Threshold: 3
               Beacon Required By:
               Uplinks: vmnic3, vmnic2, vmnic1, vmnic0
               Portgroups: VM Network, Management Network
        
        The **both** means CDP is now running, and the `lldpd` dameon on
        Cumulus RMP is capable of "seeing" CDP devices.

2.  After the next CDP interval, the Cumulus RMP box will pick up the
    interface via the `lldp` daemon:
    
        cumulus@switch$ lldpctl show neighbor swp49
        -------------------------------------------------------------------------------
        LLDP neighbors:
        -------------------------------------------------------------------------------
        Interface:    swp49, via: CDPv2, RID: 6, Time: 0 day, 00:34:58
          Chassis:
            ChassisID:    local NX-1050-A
            SysName:      NX-1050-A
            SysDescr:     Releasebuild-2494585 running on VMware ESX
            MgmtIP:       0.0.0.0
            Capability:   Bridge, on
          Port:
            PortID:       ifname vmnic2
            PortDescr:    vmnic2
        -------------------------------------------------------------------------------

3.  Use `netshow` to look at `lldp` information:
    
        cumulus@switch$ netshow lldp
        --------------------------------------------------------------------
        To view the legend,  rerun "netshow" cmd with the  "--legend" option
        --------------------------------------------------------------------
        Local Port    Speed        Mode             Remote Port    Remote  Host       Summary
        ------------  -----------  ---------  ----  -------------  -----------------  --------------------------
        eth0          1G           Mgmt       ====  swp32          swoob.vsokt.local  IP: 192.168.0.111/24(DHCP)
        swp49         10G(SFP+)    Access/L2  ====  vmnic2         NX-1050-A          Untagged: br-ntnx
        swp50         10G(SFP+)    Access/L2  ====  vmnic2         NX-1050-B          Untagged: br-ntnx
        swp51         10G(SFP+)    Access/L2  ====  vmnic2         NX-1050-C          Untagged: br-ntnx
        swp52         10G(SFP+)    Access/L2  ====  vmnic2         NX-1050-D          Untagged: br-ntnx

## Enabling LLDP/CDP on Nutanix Acropolis (Hypervisor on Nutanix Acropolis)</span>

[Nutanix Acropolis](http://www.nutanix.com/products/acropolis/) is an
alternate hypervisor that Nutanix supports. Acropolis Hypervisor uses
the yum packaging system and is capable of installing normal Linux lldp
daemons to operating just like Cumulus RMP. LLDP should be enabled for
each interface on the host. Refer to
<https://community.mellanox.com/docs/DOC-1522> for setup instructions.

## snmpwalk the Switch from Another Linux Device</span>

One of the most important ways to troubleshoot is to snmpwalk the switch
from another Linux device that can reach the switch running Cumulus RMP.
For this demonstration, another switch running Cumulus RMP within the
network is used.

1.  Open `/etc/apt/sources.list` in an editor.

2.  Add the following line, and save the file:
    
        deb http://ftp.us.debian.org/debian/ jessie main non-free

3.  Update the switch:
    
        cumulus@switch2$ sudo apt-get update

4.  Install the snmp and snmp-mibs-downloader packages:
    
        cumulus@switch2$ sudo apt-get install snmp snmp-mibs-downloader

5.  Verify that the "mibs :" line is commented out in
    `/etc/snmp/snmp.conf`:
    
        #
        # As the snmp packages come without MIB files due to license reasons, loading
        # of MIBs is disabled by default. If you added the MIBs you can reenable
        # loading them by commenting out the following line.
        #mibs :

6.  Perform an snmpwalk on the switch. The switch running snmpd in the
    demonstration is using IP address 192.168.0.111. It is possible to
    snmpwalk the switch from itself, following these instructions,
    ruling out an snmp problem vs networking problem.
    
        cumulus@switch2$ snmpwalk -c public -v2c 192.168.0.111
    
    ##### Output Examples</span>
    
        IF-MIB::ifPhysAddress.2 = STRING: 74:e6:e2:f5:a2:80
        IF-MIB::ifPhysAddress.3 = STRING: 0:e0:ec:25:b8:54
        IF-MIB::ifPhysAddress.4 = STRING: 74:e6:e2:f5:a2:81
        IF-MIB::ifPhysAddress.5 = STRING: 74:e6:e2:f5:a2:82
        IF-MIB::ifPhysAddress.6 = STRING: 74:e6:e2:f5:a2:83
        IF-MIB::ifPhysAddress.7 = STRING: 74:e6:e2:f5:a2:84
        IF-MIB::ifPhysAddress.8 = STRING: 74:e6:e2:f5:a2:85
        IF-MIB::ifPhysAddress.9 = STRING: 74:e6:e2:f5:a2:86
        IF-MIB::ifPhysAddress.10 = STRING: 74:e6:e2:f5:a2:87
        IF-MIB::ifPhysAddress.11 = STRING: 74:e6:e2:f5:a2:88
        IF-MIB::ifPhysAddress.12 = STRING: 74:e6:e2:f5:a2:89
        IF-MIB::ifPhysAddress.13 = STRING: 74:e6:e2:f5:a2:8a
        IF-MIB::ifPhysAddress.14 = STRING: 74:e6:e2:f5:a2:8b
        IF-MIB::ifPhysAddress.15 = STRING: 74:e6:e2:f5:a2:8c
        IF-MIB::ifPhysAddress.16 = STRING: 74:e6:e2:f5:a2:8d
        IF-MIB::ifPhysAddress.17 = STRING: 74:e6:e2:f5:a2:8e
        IF-MIB::ifPhysAddress.18 = STRING: 74:e6:e2:f5:a2:8f
        IF-MIB::ifPhysAddress.19 = STRING: 74:e6:e2:f5:a2:90

Any information gathered here should verify that snmpd is running
correctly on the Cumulus RMP side, reducing locations where a problem
may reside.

### Troubleshooting Tips Table for snmp walks</span>

<table>
<colgroup>
<col style="width: 33%" />
<col style="width: 33%" />
<col style="width: 33%" />
</colgroup>
<thead>
<tr class="header">
<th><p>Run snmpwalk from</p></th>
<th><p>If it works</p></th>
<th><p>If it does not work</p></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p><strong>switch</strong> (switch to be monitored)</p></td>
<td><p>snmpd is serving information correctly<br />
Problem resides somewhere else (e.g. network connectivity, Prism misconfiguration)</p></td>
<td><p>Is snmpd misconfigured or installed incorrectly?</p></td>
</tr>
<tr class="even">
<td><p><strong>switch2</strong> (another Cumulus RMP switch in the network)</p></td>
<td><p>snmpd is serving information correctly and network reachability works between <strong>switch</strong> and <strong>switch2</strong><br />
Problems resides somewhere else (e.g. can Prism reach <strong>switch</strong>, Prism misconfiguration)</p></td>
<td><p>Network connectivity is not able to grab information?<br />
Is there an iptables rule blocking? Is the snmp walk being run correctly?</p></td>
</tr>
<tr class="odd">
<td><p><strong>Nutanix Prism CLI</strong> (ssh to the cluster IP address)</p></td>
<td><p>snmpd is serving information correctly and network reachability works between <strong>switch</strong> and the <strong>Nutanix Appliance</strong> Problems resides somewhere else (e.g. The GUI might be misconfigured)</p></td>
<td><p>Is the right community name being used in the GUI? Is snmp v2c being used?</p></td>
</tr>
</tbody>
</table>

## Troubleshooting Connections without LLDP or CDP</span>

1.  Find the MAC address information in the Prism GUI, located in:
    **Hardware -\> Table -\> Host -\> Host NICs**

2.  Select a MAC address to troubleshoot (e.g. 0c:c4:7a:09:a2:43
    represents vmnic0 which is tied to NX-1050-A).

3.  List out all the MAC addresses associated to the bridge:
    
        cumulus@switch$ brctl showmacs br-ntnx
        port name mac addr     vlan    is local?   ageing timer
        swp9      00:02:00:00:00:06 0   no        66.94
        swp52     00:0c:29:3e:32:12  0   no         2.73
        swp49     00:0c:29:5a:f4:7f  0   no         2.73
        swp51     00:0c:29:6f:e1:e4  0   no         2.73
        swp49     00:0c:29:74:0c:ee  0   no         2.73
        swp50     00:0c:29:a9:36:91  0   no         2.73
        swp9      08:9e:01:f8:8f:0c  0   no        13.56
        swp9      08:9e:01:f8:8f:35  0   no         2.73
        swp4      0c:c4:7a:09:9e:d4  0   no        24.05
        swp1      0c:c4:7a:09:9f:8e  0   no        13.56
        swp3      0c:c4:7a:09:9f:93  0   no        13.56
        swp2      0c:c4:7a:09:9f:95  0   no        24.05
        swp52     0c:c4:7a:09:a0:c1  0   no         2.73
        swp51     0c:c4:7a:09:a2:35  0   no         2.73
        swp49     0c:c4:7a:09:a2:43  0   no         2.73
        swp9      44:38:39:00:82:04  0   no         2.73
        swp9      74:e6:e2:f5:a2:80  0   no         2.73
        swp1      74:e6:e2:f5:a2:81  0   yes        0.00
        swp2      74:e6:e2:f5:a2:82  0   yes        0.00
        swp3      74:e6:e2:f5:a2:83  0   yes        0.00
        swp4      74:e6:e2:f5:a2:84  0   yes        0.00
        swp5      74:e6:e2:f5:a2:85  0   yes        0.00
        swp6      74:e6:e2:f5:a2:86  0   yes        0.00
        swp7      74:e6:e2:f5:a2:87  0   yes        0.00
        swp8      74:e6:e2:f5:a2:88  0   yes        0.00
        swp9      74:e6:e2:f5:a2:89  0   yes        0.00
        swp10     74:e6:e2:f5:a2:8a  0   yes        0.00
        swp49     74:e6:e2:f5:a2:b1  0   yes        0.00
        swp50     74:e6:e2:f5:a2:b2  0   yes        0.00
        swp51     74:e6:e2:f5:a2:b3  0   yes        0.00
        swp52     74:e6:e2:f5:a2:b4  0   yes        0.00
        swp9      8e:0f:73:1b:f8:24  0   no         2.73
        swp9      c8:1f:66:ba:60:cf  0   no        66.94
    
    Alternatively, you can use grep:p
    
        cumulus@switch$ brctl showmacs br-ntnx | grep 0c:c4:7a:09:a2:43
        swp49     0c:c4:7a:09:a2:43   0   no         4.58
        cumulus@switch$
    
    vmnic1 is now hooked up to swp49. This matches what is seen in lldp:
    
        cumulus@switch$ lldpctl show neighbor swp49
        -------------------------------------------------------------------------------
        LLDP neighbors:
        -------------------------------------------------------------------------------
        Interface:    swp49, via: CDPv2, RID: 6, Time: 0 day, 01:11:12
          Chassis:
            ChassisID:    local NX-1050-A
            SysName:      NX-1050-A
            SysDescr:     Releasebuild-2494585 running on VMware ESX
            MgmtIP:       0.0.0.0
            Capability:   Bridge, on
          Port:
            PortID:       ifname vmnic2
            PortDescr:    vmnic2
        -------------------------------------------------------------------------------
        cumulus@switch$

## SNMP Traps</span>

### snmptrapd.conf</span>

The Net-SNMP trap daemon configuration file, `/etc/snmptrapd.conf`, is
used to configure how incoming traps should be processed. For more
information about specific configuration options within the file, run
the following command in a terminal:

    cumulus@switch:~$ man 5 snmptrapd.conf
      ###############################################################################

    #
    # EXAMPLE-trap.conf:
    #   An example configuration file for configuring the Net-SNMP snmptrapd agent.
    #
    ###############################################################################
    #
    # This file is intended to only be an example.  If, however, you want
    # to use it, it should be placed in /etc/snmp/snmptrapd.conf.
    # When the snmptrapd agent starts up, this is where it will look for it.
    #
    # All lines beginning with a '#' are comments and are intended for you
    # to read.  All other lines are configuration commands for the agent.
    
    #
    # PLEASE: read the snmptrapd.conf(5) manual page as well!
    #
    snmpTrapdAddr localhost
    forward default {{global['snmp_server']}}

### Generating Event Notification Traps</span>

The Net-SNMP agent provides a method to generate SNMP trap events, via
the Distributed Management (DisMan) Event MIB, for various system
events, including linkup/down, exceeding the temperature sensor
threshold, CPU load, or memory threshold, or other SNMP MIBs.

#### Enabling MIB to OID Translation</span>

MIB names can be used instead of OIDs, by installing the
`snmp-mibs-downloader`, to download SNMP MIBs to the switch prior to
enabling traps. This greatly improves the readability of the
`snmpd.conf` file.

1.  Open `/etc/apt/sources.list` in a text editor.

2.  Add the `non-free` repository, and save the file:
    
        cumulus@switch:~$ deb http://ftp.us.debian.org/debian/ jessie main non-free

3.  Update the switch:
    
        cumulus@switch:~$ apt-get update

4.  Install the `snmp-mibs-downloader`:
    
        apt-get snmp-mibs-downloader

5.  Open the `/etc/snmp/snmp.conf` file to verify that the `mibs :` line
    is commented out:
    
        #
        # As the snmp packages come without MIB files due to license reasons, loading
        # of MIBs is disabled by default. If you added the MIBs you can reenable
        # loading them by commenting out the following line.
        #mibs :

6.  Open the `/etc/default/snmpd` file to verify that the `export MIBS=`
    line is commented out:
    
        # This file controls the activity of snmpd and snmptrapd
        
        # Don't load any MIBs by default.
        # You might comment this lines once you have the MIBs Downloaded.
        #export MIBS=

7.  Once the configuration has been confirmed, remove or comment out the
    `non-free` repository in `/etc/apt/sources.list`.
    
        #deb http://ftp.us.debian.org/debian/ jessie main non-free

#### Configuring Trap Events</span>

The following configurations should be made in `/etc/snmp/snmp.conf`, in
order to enable specific types of traps. Once configured, restart the
`snmpd` service to apply the changes.

    cumulus@switch:~$ sudo systemctl restart snmpd.service

**Defining Access Credentials**

An SNMPv3 username is required to authorize the DisMan service. The
example code below uses `cumulusUser` as the username.

    createUser cumulusUser
    iquerySecName cumulusUser
    rouser cumulusUser

**Defining Trap Receivers**

The example code below creates a trap receiver that is capable of
receiving SNMPv2 traps.

    trap2sink 192.168.1.1 public

{{%notice note%}}

Although the traps are sent to an SNMPV2 receiver, the SNMPv3 user is
still required.

{{%/notice%}}

{{%notice note%}}

It is possible to define multiple trap receivers, and to use the domain
name instead of IP address in the `trap2sink` directive.

{{%/notice%}}

**Configuring LinkUp/Down Notifications**

The `linkUpDownNotifications` directive is used to configure linkup/down
notifications when the operational status of the link changes.

    linkUpDownNotifications yes

{{%notice note%}}

The default frequency for checking link up/down is 60 seconds. The
default frequency can be changed using the `monitor` directive directly
instead of the `linkUpDownNotifications` directive. See `man snmpd.conf`
for details.

{{%/notice%}}

**Configuring Temperature Notifications**

Temperature sensor information for each available sensor is maintained
in the the lmSensors MIB. Each platform may contain a different number
of temperature sensors. The example below generates a trap event when
any temperature sensors exceeds a threshold of 68 degrees (centigrade).
It monitors each `lmTempSensorsValue`. When the threshold value is
checked and exceeds the `lmTempSensorsValue`, a trap is generated. The
`-o lmTempSenesorsDevice` option is used to instruct SNMP to also
include the lmTempSensorsDevice MIB in the generated trap. The default
frequency for the `monitor` directive is 600 seconds. The default
frequency may be changed using the `-r` option.:

    monitor lmTemSensor -o lmTempSensorsDevice lmTempSensorsValue > 68000

Alternatively, temperature sensors may be monitored individually. To
monitor the sensors individually, first use the `sensors` command to
determine which sensors are available to be monitored on the platform.

    #sensors
      
    CY8C3245-i2c-4-2e
    Adapter: i2c-0-mux (chan_id 2)
    fan5: 7006 RPM (min = 2500 RPM, max = 23000 RPM)
    fan6: 6955 RPM (min = 2500 RPM, max = 23000 RPM)
    fan7: 6799 RPM (min = 2500 RPM, max = 23000 RPM)
    fan8: 6750 RPM (min = 2500 RPM, max = 23000 RPM)
    temp1: +34.0 C (high = +68.0 C)
    temp2: +28.0 C (high = +68.0 C)
    temp3: +33.0 C (high = +68.0 C)
    temp4: +31.0 C (high = +68.0 C)
    temp5: +23.0 C (high = +68.0 C)

Configure a `monitor` command for the specific sensor using the `-I`
option. The `-I` option indicates that the monitored expression is
applied to a single instance. In this example, there are five
temperature sensors available. The following monitor directive can be
used to monitor only temperature sensor three at five minute intervals.

    monitor -I -r 300 lmTemSensor3 -o lmTempSensorsDevice.3 lmTempSensorsValue.3 > 68000

**Configuring Free Memory Notifications**

You can monitor free memory using the following directives. The example
below generates a trap when free memory drops below 1,000,000KB. The
free memory trap also includes the amount of total real memory:

    monitor MemFreeTotal -o memTotalReal memTotalFree <  1000000

**Configuring Processor Load Notifications**

To monitor CPU load for 1, 5 or 15 minute intervals, use the `load`
directive in conjunction with the `monitor` directive. The following
example will generate a trap when the 1 minute interval reaches 12%, the
5 minute interval reaches 10% or the 15 minute interval reaches 5%.

    load 12 10 5
    monitor -r 60 -o laNames -o laErrMessage "laTable" laErrorFlag !=0

**Configuring Disk Utilization Notifications**

To monitor disk utilization for all disks, use the `includeAllDisks`
directive in conjunction with the `monitor` directive. The example code
below generates a trap when a disk is 99% full:

    includeAllDisks 1%
    monitor -r 60 -o dskPath -o DiskErrMsg "dskTable" diskErrorFlag !=0

**Configuring Authentication Notifications**

To generate authentication failure traps, use the `authtrapenable`
directive:

    authtrapenable 1

## <span id="src-5118703_SNMPMonitoring-supported_mibs" class="confluence-anchor-link"></span>Supported MIBs</span>

Below are the MIBs supported by Cumulus RMP, as well as suggested uses
for them. The overall Cumulus RMP MIB is defined in
`/usr/share/snmp/Cumulus-Snmp-MIB.txt`.

| MIB Name                                                                                                   | Suggested Uses                                                                                                                                                                                                                                                                                                                                                                                                   |
| ---------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| BRIDGE and Q-BRIDGE                                                                                        | The dot1dBasePortEntry and dot1dBasePortIfIndex tables in the BRIDGE-MIB and dot1qBase, dot1qFdbEntry, dot1qTpFdbEntry, dot1qTpFdbStatus, and the dot1qVlanStaticName tables in the Q-BRIDGE-MIB tables. You must uncomment the `bridge_pp.py pass_persist` script in `/etc/snmp/snmpd.conf`.                                                                                                                    |
| CUMULUS-COUNTERS-MIB                                                                                       | Discard counters: Cumulus RMP also includes its own counters MIB, defined in `/usr/share/snmp/Cumulus-Counters-MIB.txt`. It has the OID `.1.3.6.1.4.1.40310.2`                                                                                                                                                                                                                                                   |
| CUMULUS-RESOURCE-QUERY-MIB                                                                                 | Cumulus RMP includes its own resource utilization MIB, which is similar to using [cl-resource-query](/cumulus-rmp-30/Monitoring-and-Troubleshooting/Resource-Diagnostics-Using-cl-resource-query). It monitors L3 entries by host, route, nexthops, ECMP groups and L2 MAC/BDPU entries. The MIB is defined in `/usr/share/snmp/Cumulus-Resource-Query-MIB.txt`, and has the OID `.1.3.6.1.4.1.40310.1.` |
| DISMAN-EVENT                                                                                               | Trap monitoring                                                                                                                                                                                                                                                                                                                                                                                                  |
| ENTITY                                                                                                     | From RFC 4133, the temperature sensors, fan sensors, power sensors, and ports are covered.                                                                                                                                                                                                                                                                                                                       |
| ENTITY-SENSOR                                                                                              | Physical sensor information (temperature, fan, and power supply) from RFC 3433.                                                                                                                                                                                                                                                                                                                                  |
| HOST-RESOURCES                                                                                             | Users, storage, interfaces, process info, run parameters                                                                                                                                                                                                                                                                                                                                                         |
| [IF-MIB](http://net-snmp.sourceforge.net/docs/mibs/interfaces.html)                                        | Interface description, type, MTU, speed, MAC, admin, operation status, counters                                                                                                                                                                                                                                                                                                                                  |
| [IP (includes ICMP)](http://net-snmp.sourceforge.net/docs/mibs/ip.html)                                    | IPv4, IPv4 addresses, counters, netmasks                                                                                                                                                                                                                                                                                                                                                                         |
| IPv6                                                                                                       | IPv6 counters                                                                                                                                                                                                                                                                                                                                                                                                    |
| IP-FORWARD                                                                                                 | IP routing table                                                                                                                                                                                                                                                                                                                                                                                                 |
| [LLDP](http://www.mibdepot.com/cgi-bin/getmib3.cgi?i=1&n=LLDP-MIB&r=cisco&f=LLDP-MIB-V1SMI.my&v=v1&t=tree) | L2 neighbor info from `lldpd` (note, you need to [enable the SNMP subagent](/cumulus-rmp-30/Layer-1-and-Layer-2-Features/Link-Layer-Discovery-Protocol/#enabling-the-snmp-subagent-in-lldp-span) in LLDP). `lldpd` needs to be started with the `-x` option to enable connectivity to `snmpd` (AgentX).                                                                                                                                               |
| [LM-SENSORS MIB](http://support.ipmonitor.com/mibs_byoidtree.aspx?oid=.1.3.6.1.4.1.2021.13.16)             | Fan speed, temperature sensor values, voltages. This is deprecated since the ENTITY-SENSOR MIB has been added.                                                                                                                                                                                                                                                                                                   |
| NET-SNMP-AGENT                                                                                             | Agent timers, user, group config                                                                                                                                                                                                                                                                                                                                                                                 |
| NET-SNMP-EXTEND                                                                                            | Agent timers, user, group config                                                                                                                                                                                                                                                                                                                                                                                 |
| [NET-SNMP-EXTEND-MIB](http://net-snmp.sourceforge.net/docs/mibs/netSnmpExtendMIB.html)                     | (See also [this knowledge base article](https://support.cumulusnetworks.com/hc/en-us/articles/204507848) on extending NET-SNMP in Cumulus RMP to include data from power supplies, fans and temperature sensors.)                                                                                                                                                                                                |
| NET-SNMP-VACM                                                                                              | Agent timers, user, group config                                                                                                                                                                                                                                                                                                                                                                                 |
| [NOTIFICATION-LOG](http://www.net-snmp.org/docs/mibs/notificationLogMIB.html)                              | Local logging                                                                                                                                                                                                                                                                                                                                                                                                    |
| [SNMP-FRAMEWORK](http://net-snmp.sourceforge.net/docs/mibs/snmpFrameworkMIB.html)                          | Users, access                                                                                                                                                                                                                                                                                                                                                                                                    |
| [SNMP-MPD](http://net-snmp.sourceforge.net/docs/mibs/snmpMPDMIB.html)                                      | Users, access                                                                                                                                                                                                                                                                                                                                                                                                    |
| [SNMP-TARGET](http://www.net-snmp.org/docs/mibs/snmpTargetMIB.html)                                        |                                                                                                                                                                                                                                                                                                                                                                                                                  |
| [SNMP-USER-BASED-SM](http://net-snmp.sourceforge.net/docs/mibs/snmpUsmMIB.html)                            | Users, access                                                                                                                                                                                                                                                                                                                                                                                                    |
| [SNMP-VIEW-BASED-ACM](http://net-snmp.sourceforge.net/docs/mibs/snmpVacmMIB.html)                          | Users, access                                                                                                                                                                                                                                                                                                                                                                                                    |
| [SNMPv2](http://net-snmp.sourceforge.net/docs/mibs/snmpMIB.html)                                           | SNMP counters (For information on exposing CPU and memory information via SNMP, see this [knowledge base article](https://support.cumulusnetworks.com/hc/en-us/articles/203922988).)                                                                                                                                                                                                                             |
| [TCP](http://net-snmp.sourceforge.net/docs/mibs/tcp.html)                                                  | TCP related information                                                                                                                                                                                                                                                                                                                                                                                          |
| [UCD-SNMP](http://www.net-snmp.org/docs/mibs/UCD-SNMP-MIB.txt)                                             | System memory, load, CPU, disk IO                                                                                                                                                                                                                                                                                                                                                                                |
| [UDP](http://net-snmp.sourceforge.net/docs/mibs/udp.html)                                                  | UDP related information                                                                                                                                                                                                                                                                                                                                                                                          |

{{%notice note%}}

The ENTITY MIB does not currently show the chassis information in
Cumulus RMP 3.0.

{{%/notice%}}

