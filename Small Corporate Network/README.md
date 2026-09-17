Small	Corporate	Network	—	Packet	Tracer
Project
A	simulated	small	corporate	network	built	in	Cisco	Packet	Tracer,	covering	VLAN	segmentation,
inter-VLAN	routing,	DHCP,	dual-WAN	redundancy,	firewall	NAT/ACLs,	and	wireless	security.
Topology	Overview
Core:	3560-24PS	Multilayer	Switch	(inter-VLAN	routing)
Access	switches	(5x	2960-24TT):	Server	Room,	Guest	Network,	IT	Department,	Corporate
Network,	Office	Users
Firewall:	Cisco	ASA	5506-X	(NAT,	guest	isolation	ACL)
WAN:	Dual	1941	routers	(primary	+	backup)	for	redundancy
Wireless:	Corporate	Wi-Fi	(WPA2-PSK)	and	Guest	Wi-Fi	APs
VLANs
VLAN
Name
Subnet
10
IT
10.10.10.0/24
20
Office
10.10.20.0/24
30
Servers
10.10.30.0/24
40
Corp_WiFi
10.10.40.0/24
50
Guest_WiFi
10.10.50.0/24
60
Voice
10.10.60.0/24
Files
Small_cooporate_network.pkt	—	the	Packet	Tracer	project	file
docs/VLAN_Trunk_Config_Rebuild.md	—	VLAN/trunk	configuration	for	all	switches
docs/Full_Network_Buildout.md	—	IP	addressing,	routing,	DHCP,	WAN,	firewall,	and	wireless
configuration
Highlights	/	Lessons	Learned
Rebuilt	VLAN	and	trunk	configuration	from	a	clean	slate	across	6	switches
Resolved	native	VLAN	mismatches	between	trunk	links
Implemented	inter-VLAN	routing	via	SVIs	on	the	core	switch
Configured	DHCP	relay	(
ip	helper-address )	across	VLANs	to	a	centralized	DHCP	server
Set	up	dual-WAN	routing	with	a	primary	and	backup	path	through	the	firewall
Applied	a	guest	network	isolation	ACL	on	the	ASA	firewall
Upgraded	wireless	security	from	WEP	to	WPA2-PSK
