	OpenFlow on ProCurve Switches
	-----------------------------

	This file describes how to use the OpenFlow module on the
ProCurve Switch J9088A.

Legal stuff
-----------
	The OpenFlow module is an *EXPERIMENTAL* feature. It was
originally contributed and implemented by HP Labs for the OpenFlow
project.  It is not an official HP Networking product, and is not
supported through normal support channels.

	License and distribution of the HP Networking firmware 
including the OpenFlow module is handled separately from production
firmware for the same product line.  The use of this firmware
requires a signed software license agreement from HP Networking,
and the firmware can not be redistributed. Please contact HP
Networking for details on firmware distribution and any licensing
issue.

Table of Contents:
------------------

1. Status and Capabilities
   a. Module status
   b. Change Log
   c. Known Bugs
   d. Limitations
   e. Hardware acceleration limitations

2. User Manual
   a. Flashing the firmware
   b. OpenFlow on a VLAN (VLAN virtualization)
   c. VLAN configuration
   d. Port configuration
   e. OpenFlow configuration
   f. Controller connection
   g. OpenFlow interfaces
   h. Rate limiters configuration
   i. Show configuration
   j. Show OpenFlow flows
   k. Show OpenFlow rules
   l. Configuration via SNMP
   m. Configuration commands (CLI, SNMP)
   n. Aggregation mode configuration

3. HPL Extensions
   a. Priority queues
   b. Per-flow rate limiters

4. Final words


*********************** STATUS & CAPABILITIES ***********************

Module status
-------------
	Current OpenFlow version : Current version : v2.02e (K.15.5.5001)

	The OpenFlow module is included in the ProCurve firmware for
the Switch J9088A (54xxxzl, 82xxxzl, 3500, 6200, 6600).

     Current OpenFlow module has been implemented in
firmware revision K.15.05. Other revisions are not available.

     Version v2.02 of the OpenFlow module implements OpenFlow 
protocol version 1.0.  A prior version of this module, v1.09, 
implements OpenFlow 0.8.9, and is available in older firmware
images. Other versions of OpenFlow are not available.

	The current module has seen limited testing and does not yet
pass the conformance tests. The current implementation has various
limitations (see below).

New in release K.15.5.5001
--------------------
	o Fixed a bug in the re-application of non-volatile OpenFlow
     configuration after a switch re-boot.

New in release K.15.5.5000
--------------------
      o Support for Version 2 chassis blades (model numbers
        9534, 9535).

      o Support for inclusion of OpenFlow configuration commands
        in the "show config" display.

      o Addition of a "no openflow" configuration negation 
        command.

      o Addition of a "show openflow flows" status display.

      o Addition of a "show openflow rules" status display.

Known Bugs
----------
	o Some parts of the OpenFlow V1.0 specification are not yet
      fully implemented
	o LAG/Trunk are not supported
	o ProCurve Mesh is not supported
	o QinQ has not been tested

Limitations
-----------
	o OpenFlow 0.8.2 - SSL controller connection : not supported
	o OpenFlow 0.8.9 - Explicit Handling of IP Fragments :
	  OFPC_IP_REASM is not supported.
	o OpenFlow 0.8.9 - 802.1D Spanning Tree : OpenFlow does not
	  allow full interaction with the switch Spanning Tree, OpenFlow is
	  confined to the switch Spanning Tree.
	o OpenFlow 0.9 - Emergency Flow Cache : not supported.
	o OpenFlow 0.9 - Barrier Command : not supported.
	o OpenFlow 1.0 - Slicing : not implemented. The hardware can
	  not support the slicing specification. We provide our own QoS API
	  instead.

Hardware acceleration limitations
---------------------------------
	All flows can not be instantiated in hardware, because of hardware
limitations.  Flows not instantiated in hardware are processed in
software, which is a slow path.

	The full set of OpenFlow features is available for flows
that are processed in software without any limitations.

	For a rule to be instantiated in hardware :
		o The flow must be IP (L2 type == 0x800)
		o The main action is output to a single port or drop
		o optional actions : set pcp, set ToS & rate limit
	Following types of rules will be processed in software:
		o output to multiple ports
		o all header rewrites except ToS & PCP
		o non-IP
		o output to OFPP_IN_PORT 
		o output to OFPP_FLOOD

	When a rule is instantiated in hardware, following fields of
the predicate are ignored:
		o dl_src is ignored
		o dl_dst is ignored
		o dl_vlan_pcp is ignored
		o flow_mod priority is ignored (wildcard flows)

	When a rule is instantiated in hardware, full statistics are
no longer available for that rule. The following restrictions apply :
		o byte_count is not available (software byte count only)
		o statistics are updated at a large period interval
		o flow_duration is larger than actual flow duration

Summary of what works in hardware and what does not:

Header fields:
Ingress Port      - matched
Ether source      - not matched; ignored
Ether dst         - not matched; ignored 
Ether type        - has to be 0x800
VLAN id           - forced to the Openflow instance vlan in the vlan 
                    virtualization mode; matched straight in the 
                    aggregation mode.
VLAN priority     - not matched; ignored
IP src            - matched
IP dst            - matched
IP proto          - matched
IP ToS bits       - matched
TCP/ UDP src port - matched
TCP/ UDP dst port - matched

Actions:
Forward           - only one physical or logical port
Forward ALL       - not supported
Forward CONTROLLER- not supported
Forward LOCAL     - not supported
Forward TABLE     - not supported
Forward IN_PORT   - not supported
Forward NORMAL    - supported
Forward FLOOD     - not supported
enqueue           - not supported (use HP QoS extensions instead)
drop              - supported
set vlan-id       - not supported
set vlan-priority - supported
strip vlan        - not supported
modify eth src    - not supported
modify eth dst    - not supported
modify ipv4 src   - not supported
modify ipv4 dst   - not supported
modify ipv4 ToS   - supported
modify tcp srcport- not supported
modify tcp dstport- not supported

Per-flow counters:
Received Packets  - maintained correctly
Received Bytes    - not maintained correctly
Duration (sec)    - software maintains this
Duration (nsec)   - software maintains this


**************************** USER MANUAL ****************************

To flash the firmware
----------------

	The switch has two firmware banks, primary and secondary. We
advise to always keep a tested version of an original HP Networking
firmware in the primary bank, and to only put OpenFlow enabled
firmware in the second bank. At boot, it is possible to select which
firmware to use from the serial console (9600 baud).

	Only HP Networking firmware K.13.58 and later can flash the
OpenFlow enabled firmware, earlier firmware will return an error. We
therefore recommend to use one of those firmwares as primary.

	Firmware upload can be done via the console of the switch,
either on the serial console or via telnet/ssh.

	The OpenFlow enabled firmware needs to be added on a TFTP
server accessible from the switch.

	To flash the OpenFlow enabled firmware :
		> copy tftp flash 10.10.10.1 K_15_05_exp_openflow_2.swi secondary
	To boot the firmware :
		> boot system flash secondary

OpenFlow on a VLAN (VLAN virtualization)
------------------
	The default operating mode of the OpenFlow module is VLAN
virtualization. A number of VLANs are designated as OpenFlow
instances. Each of these OpenFlow VLANs is independent and has its own
OpenFlow configuration and controller connection. Each OpenFlow
instance runs logically within the corresponding VLAN.

	Normal operation also requires some VLANs which are not
managed by OpenFlow (at least one). The non-OpenFlow VLANs are used to
run the OpenFlow controller connections and to configure the OpenFlow
instances via SNMP. Non-OpenFlow instances can also be used for any
traffic that is not supposed to be managed by OpenFlow.

	Typical networks use VLAN 1 as the management VLAN. It is
probably a good idea to never use OpenFlow on VLAN 1, and to never
associate an OpenFlow configuration with VLAN 1.

VLAN configuration
------------------
	Each OpenFlow instance will only act on the traffic within a
specific VLAN. This means that you need to create a VLAN and assign
some interfaces/ports to it.

	Please refer to the section "Static Virtual LANs (VLANs)" in
the HP Networking manual "Advanced Traffic Management Guide" for
more information about configuring VLANs.

	To display the current VLANS defined :
		> show vlan
	To display the configuration and ports of VLAN 10 :
		> show vlan 10
	To create a VLAN :
		> vlan 10 name openflow
	To add ports to a VLAN :
		> vlan 10 untagged A14,B7
	To remove ports from a VLAN :
		> vlan 10 untagged A14,B7

Port Configuration
------------------
	Ports are assigned to VLAN (see VLAN configuration), and ports
can not be assigned directly to an OpenFlow instance. A port can be
part of multiple VLANs when using tagged mode, which means that a 
port can be part of multiple OpenFlow instances and non-OpenFlow 
VLANs.

	Please refer to the section "Port Status and Configuration" in
the HP Networking manual "Management and Configuration Guide" for 
more information about configuring interfaces.

	To display the status of all ports :
		> show interfaces brief
	To display the configuration of all ports:
		> show interfaces config
	To display the port statistics counters :
		> show interfaces
	To display the port statistics counters interactively :
		> show interfaces display
	To display the VLANs associated with a port
		> show vlan port A14

OpenFlow configuration
----------------------
	Configuration is mainly done on the switch command line. To
change the configuration, it is necessary to be in the configuration
mode.

	There can be multiple OpenFlow instances, and each instance is
bound to a specific VLAN. Therefore, the configuration of OpenFlow
is done on per VLAN basis.

	NOTE: Ensure that the VLAN used in the commands below is
already defined (see VLAN configuration on how to create a VLAN or
check currently defined VLANs).

	To show the OpenFlow configuration of all VLANs :
		> show openflow
	To show the OpenFlow configuration and status for VLAN 10 :
		> show openflow 10
	To enter the configuration mode :
		> configure
	To enable/start OpenFlow on VLAN 10 :
		> vlan 10 openflow enable
	To disable/stop OpenFlow on VLAN 10 :
		> vlan 10 openflow disable
	To set the controller :
		> vlan 10 openflow controller tcp:10.11.12.1:975
	To enter the VLAN 10 context and add a passive listener for dpctl :
		> vlan 10
		> openflow listener ptcp:975
	To do everything on one line :
		> vlan 10 openflow controller tcp:10.11.12.1:975 listener ptcp:975 enable

	The switch has many configuration files.  All OpenFlow
configuration done via the CLI only affect the "running-config", which
is not persistent across reboots.  Please refer to the section "Switch
Memory and Configuration" in the HP Networking manual "Management and
Configuration Guide" for more information about managing configuration
files.

	The simplest way to have persistent configuration is to save
the current "running-config" in the "startup-config".
	To save the configuration for the next reboot :
		> write memory

	Non-OpenFlow firmware won't recognize OpenFlow parameters from
configuration files and will discard them.  When booting a non-
OpenFlow firmware, all OpenFlow configuration is removed from the
"startup-config" and need to be restored when booting an OpenFlow
firmware.

OpenFlow module configuration may be removed, without erasing
the entire switch configuration.

   To remove the OpenFlow configuration on VLAN 10:
      > no openflow vlan 10
   To remove OpenFlow configuration on aggregate VLANs:
      > no openflow aggregate_vlans

Controller connection
---------------------
	The controller traffic can not be "in-band": it can not
transit on a VLAN managed by OpenFlow, and must transit on a VLAN
not managed by OpenFlow.  On the other hand, the controller traffic
and OpenFlow traffic can transit on the same port, as long as they
use different VLANs.

	The VLAN chosen for the controller traffic depends entirely on
the IP address of the controller, and no specific configuration is
needed.  For this reason, the switch should have a proper IP
configuration, and the controller address must be part of a subnet
not on an OpenFlow VLAN.

	Please refer to the section "Configuring IP Addressing" in the
HP Networking manual "Management and Configuration Guide" on how to
either manually assign an IP address to the switch or set it up to
perform DHCP queries.

OpenFlow interfaces
--------------------
	By design, the OpenFlow module user interface on the switch is
minimal, and mostly offers configuration or status information not
available through the OpenFlow protocol.  The proper way to control
and monitor the OpenFlow module is to use the OpenFlow protocol
itself.  The OpenFlow module has two OpenFlow interfaces, the
controller interface and the listener interface.

	The controller interface is an active connection to a single
controller IP address.  The controller can be monolithic, but it can
also be a proxy, such as FlowVisor, or a framework, such as NoX,
enabling multiple applications to interact with the switch via the
controller connection.
	To set the controller for an OpenFlow instance:
		> vlan 10 openflow controller tcp:10.11.12.1:975

	The listener interface is a passive connection enabling any
number of OpenFlow applications to connect directly to the switch,
but does not offer the full controller functionality.
	To add a passive listener interface :
		> vlan 10 openflow listener ptcp:975

	The most common way to debug and monitor the OpenFlow module
is to use an utility running on a Linux system and connecting remotely
to the listener socket on the switch.  The OpenFlow Reference System
software contains such an utility called "dpctl".  Open vSwitch
contains its own version of this utility called "ovs-ofctl".

	The syntax of dpctl and ovs-ofctl are identical. These four
commands expose the full state of the OpenFlow module :
		# dpctl show tcp:10.11.12.1:975
		# dpctl dump-tables tcp:10.11.12.1:975
		# dpctl dump-flows tcp:10.11.12.1:975
		# dpctl dump-ports tcp:10.11.12.1:975

	Adding a permanent flow can also be done :
		# dpctl add-flow tcp:10.11.12.1:975 "priority=32769,idle_timeout=0,ip,nw_dst=10.0.0.3,nw_src=10.0.0.2,action=output:23" 

Rate limiters configuration
---------------------------
	The OpenFlow module can be configured with rate limiters to
control OpenFlow experiments and to prevent them from consuming
significant portion of the switch resources.  It implements two rate
limiters for each instance, which have different purposes.  Each
instance has completely independent rate limiters that can be set
independently.

	The first rate limiter is the software rate limiter.  It
limits the number of packets that each line card can send to the
software path on the CPU.  It indirectly controls the number of
packet_in messages sent by the switch to the controller.  This rate
limiter is alway enabled and its default value is 100.
	To change the software rate limiter's limit :
		> vlan 10 openflow sw-rate 10000

	The second rate limiter is the hardware rate limiter.  It
controls the bandwidth for that openflow instance on each line card.
This rate limiter is disabled by default, enabling this rate limiter
disable per-flow rate limiters on that instance (see below).
	To set the hardware rate limiter :
		> vlan 10 openflow hw-rate 1000000
	To disable the hardware rate limiter :
		> vlan 10 openflow hw-rate 0

Show Configuration
------------------

     OpenFlow configuration may be extracted and saved for later
restoration on the switch (or some other switch) using the 
standard "show config" command.  The OpenFlow related display
will be formatted as follows:

 openflow
   vlan 9
      enable
      controller "tcp:172.22.18.193:6633" listener "ptcp:6633"
      exit
   exit

Show OpenFlow flows
-------------------

    The current (volatile) status of OpenFlow flows may be
displayed using the "show openflow [vlan] <vlan-id> flows"
command.

   To display the OpenFlow flow table for VLAN 9:
      > show openflow vlan 9 flows

The display is formatted as follows:

Flow 1:

 Incoming Port      : 0
 Destination MAC    : 001b21-22edd8     Source MAC         : 001b21-239552
 VLAN ID            : 0                 VLAN Priority      : 0
 Source IP          : 192.16.200.4      Destination IP     : 192.16.200.4
 IP Protocol        : IP                IP ToS bits        : 0
 Source Port        : 0                 Destination Port   : 0
 Duration           : 16s secs          Priority           : 32769
 Idle Timeout       : 0 secs            Hard Timeout       : 0 secs
 Packet Count       : 0                 Bytes Count        : 0
 Actions            : output:23

Show OpenFlow rules
-------------------

   The current (volatile) status of OpenFlow policy engine
rules may be displayed using the "show openflow [vlan] <vlan-id>
rules" command"

   To display the OpenFlow policy engine rules for VLAN 9:
       > show openflow vlan 9 rules

The display is formatted as follows:

Rule 1:

 Source IP           : 192.16.200.4/255.255.255.255
 Destination IP      : 192.168.201.3/255.255.255.255
 VLAN ID             : 9
 Protocol            : 0x0000
 Source UDP/TCP port : 0x0000
 Dest.  UDP/TCP port : 0x0000
 IP ToS bits         : 00:00
 Actions             : Forward to 002

Configuration via SNMP
----------------------
	It is also possible to configure the OpenFlow module via SNMP,
like the rest of the switch configuration. Well, that's assuming your
admin gives you the right authorization for that.

	To show the OpenFlow configuration :
		# snmpwalk -v2c -c public 10.11.12.1 iso.org.dod.internet.private.enterprises.11.2.14.11.5.1.7.1.35

	The OpenFlow MIB is currently organized as follows :
		*.1.1.2.<vlan>  -> Enable(1)/Disable(2)
		*.1.1.3.<vlan>  -> Controller

	To enable OpenFlow on VLAN 10 :
		# snmpset -v2c -c public  10.11.12.1 iso.org.dod.internet.private.enterprises.11.2.14.11.5.1.7.1.35.1.1.2.10 i 1
	To disable OpenFlow :
		# snmpset -v2c -c public  10.11.12.1 iso.org.dod.internet.private.enterprises.11.2.14.11.5.1.7.1.35.1.1.2.10 i 2

Configuration commands (CLI, SNMP)
----------------------------------
	This is the full list of commands available through CLI and
SNMP.

vlan <vlan> openflow enable		- snmp *.1.1.2.<vlan> - {1, 2}
	Enable and start the OpenFlow module.

vlan <vlan> openflow disable		- snmp *.1.1.2.<vlan> - {1, 2}
	Disable and stop the OpenFlow module.

no openflow vlan <vlan>
    Disable the OpenFlow module in the non-volatile configuration
    file store.

vlan <vlan> openflow controller		- snmp *.1.1.3.<vlan> - string
	Set the controller pseudo-URL that OpenFlow connects to.

vlan <vlan> openflow listener		- snmp *.1.1.4.<vlan> - string
	Set the listener pseudo-URL where OpenFlow receives anxilary requests.

vlan <vlan> openflow sw-rate		- snmp *.1.1.5.<vlan> - [10, 10000]
	The packet rate limit for OpenFlow's software path (packets per
	seconds per line card).

vlan <vlan> openflow backoff		- snmp *.1.1.6.<vlan> - [1, 3600]
	The maximum backoff when connecting to the controller (seconds).

vlan <vlan> openflow hw-accel		- snmp *.1.1.7.<vlan> - {1, 2}
	Enable/Disable Hardware Acceleration.

vlan <vlan> openflow hw-rate		- snmp *.1.1.8.<vlan> - int32
	The rate limit for OpenFlow's hardware path (Kilobit per seconds
	per line card).

vlan <vlan> openflow hw-refresh-max	- snmp *.1.1.9.<vlan> - [1, 3600]
	The maximum time before refreshing statistics from the hardware
	(seconds).

vlan <vlan> openflow fail-secure	- snmp *.1.1.11.<vlan> - {1, 2}
	Enable fail-secure mode, disable fail-standalone mode.

vlan <vlan> openflow second_controller	- snmp *.1.1.12.<vlan> - string
	Set the second controller pseudo-URL that OpenFlow connects to.

vlan <vlan> openflow third_controller	- snmp *.1.1.13.<vlan> - string
	Set the third controller pseudo-URL that OpenFlow connects to.

openflow aggregate_vlans enable			- snmp *.2.1.0 - {1, 2}
	Enable aggregation of OpenFlow VLANs.

openflow aggregate_vlans disable		- snmp *.2.1.0 - {1, 2}
	Disable aggregation of OpenFlow VLANs.

no openflow aggregate_vlans
   Disable aggregation of OpenFLow VLANs in the non-volatile
   configuration file store.

openflow aggregate_vlans management_vlan	- snmp *.2.2.0 - <vlan>
	Set the VlanId for the non-aggregated VLAN.

openflow aggregate_vlans configuration_vlan	- snmp *.2.3.0 - <vlan>
	Set the VlanId for the VLAN configuration to use when doing
	aggregation.

Aggregation mode configuration
------------------------------
	Aggregation is an EXPERIMENTAL feature.  Few people really need
aggregation mode, and most people are encouraged to use VLAN
virtualization and ignore this section.

	Aggregation is the opposite of VLAN virtualization, it enables
to have a single OpenFlow instance managing all the VLANs within the
switch.  It impacts the whole way OpenFlow is managed inside the
switch.  It is not possible to mix aggregation and virtualization,
which means that when aggregation is enabled, a single OpenFlow
instance can be run: the aggregated one.

	In aggregated mode, all VLANs of the switch are managed by a
single OpenFlow instance, regardless of their individual OpenFlow
configuration or lack of OpenFlow configuration. Because the OpenFlow
control traffic can not be "in-band", a specific "Management VLAN" is
defined and excluded from aggregation.  In other words, the OpenFlow
instance manages all VLANs except the "Management VLAN". The
"Management VLAN" can be used for running the OpenFlow control
traffic, but also to access the switch console or MIB, or for any
other purpose.

	The "Configuration VLAN" is only an artifact of the user
interface, it defines which VLAN instance will run the aggregation.
It just defines which OpenFlow VLAN configuration is used by the
aggregated instance.  Any VLAN number can be picked, apart from the
"Management VLAN", it won't change the overall behavior of the
switch.

	To define VLAN 1 as the Management VLAN :
		> openflow aggregate_vlans management_vlan 1
	To configure an aggregated OpenFlow :
		> openflow aggregate_vlans configuration_vlan 10
		> vlan 10 openflow controller tcp:10.11.12.1:975 listener ptcp:975
		> openflow aggregate_vlans enable
	To enable/start the aggregated OpenFlow :
		> vlan 10 openflow enable
	To show the configuration and status for the aggregated OpenFlow :
		> show openflow 10

*************************** HP EXTENSIONS ***************************

	Our OpenFlow module is the first implementation to provide QoS
support in OpenFlow.  Our QoS extensions were included in version 1.09
of the OpenFlow module, and are also available in version 2.02.

Priority queues
---------------
	Those HP Networking switches have 8 priority queues per port. The
queuing mechanism is a strict priority with guaranteed minimum
bandwidth (GMB).  The GMB parameter can be configured per port. Please
refer to the section "Port Traffic Controls" in the HP Networking
manual "Management and Configuration Guide".

	OpenFlow flows can be mapped to the priority queues using
either the VLAN PCP or the IP ToS/DSCP. The VLAN PCP offers a direct
mapping, whereas IP ToS/DSCP is indirect.  Please refer to the section
"Quality of Service (QoS)" in the HP Networking manual "Advanced 
Traffic Management Guide".

	There is no need to use the classifier feature of the switch,
the only thing needed is for the OpenFlow rule to specify the chosen
PCP or ToS as part of the action list.

	To map a flow to the third priority class :
                # dpctl add-flow tcp:10.11.12.1:975 "priority=32769,idle_timeout=0,ip,nw_dst=10.0.0.3,nw_src=10.0.0.2,action=mod_vlan_pcp:3,output:23"

Per-flow rate limiters
----------------------
	The OpenFlow module offers per flow rate limiters. Any number
of flows can be flexibly mapped to one of the rate limiters,
irrespective of their source and destination ports.

	Per-flow rate limiters are used only if the hardware rate
limiter for the instance is disabled.  To disable the hardware rate
limiter for this instance :
		> vlan 10 openflow hw-rate 0

	The use of rate limiters requires a version of ovs-ofctl which
includes our QoS extension.
        To create a rate limiter that has a limit of 512 kb/s:
		# ovs-ofctl add-limiter tcp:10.11.12.1:975 123 drop 512 kbps
	To assign a flow to the above rate limiter:
		# ovs-ofctl add-flow tcp:10.11.12.1:975 "priority=32769,idle_timeout=0,ip,nw_dst=10.0.0.3,nw_src=10.0.0.2,action=rate_limit:123,output:17"

*************************** FINAL WORDS ******************************

	For further information, support, license requests and firmware
image access requests please contact:

	Mehul Pandya  mehul.pandya@hp.com