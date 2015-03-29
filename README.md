
Si46ib9d - Setup IPv4 and IPv6 for ISC Bind9 and DHCP
-----------------------------------------------------

# Introduction

This is a script you run on a single master configuration file
( /etc/bind/si46.master ) that sets up all the real configurations
for address allocation and host naming on a typical small network,
for both IPv4 and IPv6.  The si46.master file is a bind9 forward 
map file, with additional directives in comment sections that allow 
building configuration for DHCP service for IPv4, and a 'what people 
usually want' split-horizon DNS service, showing hosts on the network 
only to other hosts on the network, with those outside the network 
seeing only one or two chosen hosts.

It does not run any services, it just defines the configurations for
the standard debian-based services.  You run all your services in the
standard way.

## What is the point?
If you want to be able to look at network traffic and have some idea who
is talking to who, you need to map the addresses to human readable names.
These are the services to do this.  This script is for people who have 
configured bind and dhcp manually before, and find it tedious.  

Using the tools as intended, you need to write the same information between 2 and 5
times, (map host name to IPv4 address, map host name to IPv6 address, add revers entries (1 for each address), add the host to MAC mapping in a dhcp configuration, for IPv4, potentially again for IPv6.  With si46ib9d, you write the 
information once, and the script creates all the other obvious copies for you.  

While this sounds like a minor convenience for IPv4, in the IPv6 world it 
becomes more important because they like to *renumber* things.  in IPv4 with 
a typical home network, you have a single public address, and then a natting 
firewall which means all your internal addresses are things you pick.  With 
IPv6,  you normally do not NAT, so the addresses on your internal network 
change because the addresses the ISP gives you change from time to time.

So when your public firewall IP changes with IPv6 (which it does a couple of times a year), it changes the *prefix* for all internal hosts, and all of the 
bind records need to be re-written.  This script, whenever it is run, just 
reads the prefix from the network, and re-writes all the IPv6 records 
correctly.  
 

# Getting Started
 
A Minimum si46.master configuration is given below:

```

  ;DNS-Domain=example.com 
  ;SOA fwgw hostmaster 

   IN NS fwgw.example.com. 

  fwgw    	IN	A	123.142.254.10 ; public

  ;DNS-Forwarders=123.142.225.1,123.142.227.2

  ;DNS-Network=192.168.10/24

  onehost IN A 3 ; MAC=52:54:00:f8:b5:26

```

As per bind documentation, a semi-colon (;) is used as a comment delimiter.  
text on a line before a any semi-colon, is a part of a normal bind forward map.
Si46 directives happen in the comments.  The directives are also case sensitive.

The first si46 directive is ;DNS-Domain, the value you set there is the 
domain you want to use for entire configuration.  Here we are building a 
configuration for example.com.  The second line is ;SOA.  This line is used 
to build the Start Of Authority, a header needed for all the forward and 
reverse zones.  Documentation of SOA records is available in bind 
documentation.  Briefly, the elements are:

```

	fwgw - the host on which the zone was created. (expands to: fwgw.example.com)
	hostmaster - admin email address  (expands to: hostmaster@example.com)

```

The rest of the SOA is given sensible defaults.  Si46 generates the serial 
number from the linux time stamp (seconds since start of epoch.)  and allows 
you to override the other settings (see si46.master.glorious for an example
with overrides.)

   IN NS fwgw.example.com. 

All zones need a name server, so use the ones declared here.
You must declare it with the full domain name (and incude the '.' at the end.)
Next is the corresponding host entry for fwgw.  The enty looks like a
a normal bind forward A record, used to define an IPv4 address.

  fwgw    	IN	A	123.142.254.10 ; public

fwgw's public IPv4 address (since it is an A record) is set to 123.142.254.10
The 'public' decoration, after the semi-colon indicates that this record
will be published to the world, and not just to internal users.

  ;DNS-Forwarders=123.142.225.1,123.142.227.2

The DNS-Forwarders directive is where you put your ISP's DNS Server 
addresses (or some other provider.) The configuration will be such that the 
server will answer requests for the local domain, refer to the Forwarders 
for names outside the domain, and cache the replies for internal users.

The next directive is to declare a network.

  ;DNS-Network=192.168.10/24
   
This means that:
	-- addresses that follow, if not complete, should be assumed to be in this network.
	-- a reverse map for this network will be created.

You may have noticed that the network in this case, is a 192.168... network.  
This is a non-routable address range.  There is an assumption that, for IPv4, 
the firewall gateway is taking care of NAT.

So with all that out of the way, here is what you need to declare machines on 
your network:

  onehost IN A 3 ; MAC=52:54:00:f8:b5:26

So there is a machine named 'onehost' whose MAC address is given.  onehost's 
IP address will be 192.168.10.3.  The DHCP configuration will be made for 
onehost to be always be given the same address, if asked.  If IPv6 is enabled, 
the MAC address can also be used to calculate the Stateless autoconfiguration 
(SLAC) address, or for dhcpv6.  

To make use of this configuration file, you need to have bind9 and 
isc-dhcp-server installed:

```

 apt-get install bind9 bindutils isc-dhcp-server

 #test to avoid overwriting real config:
 si46ib9d -r ./si46.master.simple -w `pwd`

```

for the simple example, si46ib9d creates: named.conf, named.conf.options, 
named.conf.local, dbf.internal.example.com, dbf.public.example.com, 
dbr.192.168.10, dbr.192.168.9, dbr.empty-zones, dhcpd.conf, named.conf, 
named.conf.local, named.conf.options

#do syntax check of the setup:
named-checkconf -z named.conf


Once you are happy that the configuraion looks good, copy it to the standard /etc/bind location.

cp si46.master /etc/bind

Then you can invoke si46ib9d without arguments and it will place all the config
files and zones under /etc/bind.

  twohost IN A 4 ; MAC=51:54:e0:f3:b5:26

When you want to add hosts, you just add a single line per host, as per twohost 
above.  Then re-run si46ib9d and the forward, reverse, and DHCP information will 
all be updated.  This ends the walk-through the simplest case, IPv4 only.

Dealing with DHCP and guests.

FIXME: to come, see example in si46.example.glorious


# Adding IPv6

My ISP has deployed IPv6 using the Rapid Deployment (RD) protocol.
Using DHCP extensions, the Router Advertisement Daemon (radvd) has been 
configured to announce addresses whenever the ISP link is brought up.  
Unlike IPv4, where networks inside are isolated from the outside via 
Network Address Translation,
or addresses are permanently assigned, in IPv6, there is the notion of "rapid-renumbering"
and the addresses assigned by my ISP change from time to time.

Whenever the IPv6 network address changes, there is no NAT, so all the addresses
on the internal zone have to change as well.  With si46ib9d, I supply the following
directives:

   ;DNS-Network=radvd

   trestler	IN 	A	eth1 ; public

   trestler IN AAAA 6rdif ; public

The DNS-Network directive makes the script consult /etc/radvd.conf to determine
the network prefix in effect.  Si46ib9d will assign addresses for all later host records 
in the file.  The host entries that follow, instead of having an address,
indicate the interface whose address should be used.  Whenever the program
is run,  the interface and radvd.conf file information is refreshed, and the zones
re-written completely.



Motivation for the si46ib9d script.

ISC tools are the most popular ones, but they are built for runing large corporate
sites, and their configuration methods, allow an almost infinite configurability, and
are quite intimidating in the number of decisions they ask.  Further, they have grown 
somewhat organically over time, and have gotten a bit unwieldy.  

This script uses a single data file, that looks a lot like an ISC forward 
zone file, but adds 'decorations' (directives hidden in zone file comments) to allow 
it to be used to build the entire configuration.  It is also much more
information dense, one can look at a few lines, at most a page, of configuration
and understand the general setup.  

The single data file is much simpler to maintain and is meant for editing 
by humans.  Once the domain and network decorators are set, adding
a new machine, is just a one line entry:

  debvm IN A 3 ; MAC=52:54:00:f8:b5:26

The above line will create an entry for the machine named 'debvm' with
the fixed IP address.  In a DHCP and Bind configuration for IPv4 and IPv6, 
you need, for example, to enter a hostname:

	-- in the IPv4 forward map for a domain, along with ip addr.
	-- in the reverse map same is repeated.
	-- in the IPv4 dhcp map have both hostname and address, as well as MAC.
	-- in the IPv6 fwd map stateless auto config is calculated from MAC 
	-- in the IPv6 dhcp map has both hostname and address, as well as MAC.

So you have to enter the hostname, mac, and address between three and 
six times.  This script eliminates that sort of repetitive drudgery.
by picking a particular network design and user case, and making 
some assumptions:

	-- The configuration is for a single domain.
	-- for every forward address on a defined network, 
	   make the obvious reverse record.
	-- Create two ISC-Bind 9 views:  internal, and public.  by default, 
           all host records are internal.  You need to decorate an entry with 
	   public, to have the entry added to the public view.
	-- @ (domain) records MX, TXT, NS, by default, are in all views.
	-- there are no DNS keys or updates ´update none.´ Use SSH or
	   some other mechanism to distribute updates.
	-- There is a main, flat network 
	-- currently, support RADVD - IPv6 Router Advertisement Daemon.
	-- bind9 and bind9utils are installed, as per Debian wheezy.
	   to provide all the static domains in /etc/bind/db.* (0, 127, empty, etc...)
           tool refers to these in their standard place.

These assumptions create a configuration which is likely suitable for 
many small businesses or (geeky!) homes.  If you don't want a public
view, do not run the configuration on a public facing server, and it
will not be a problem.  if a firewalling mistake grants an external user
access to the internal network, they will only get the public view,
which is still a benefit.  

The script is can be run regularly to re-create zones when using a system 
with an ISP provided dynamic address, allocted by some mechanism such as 
DHCP.  The one departure from making the si46.master file a valid forward
zone is that one can place an interface address in place of an IP (4 or 6)
address in the file, which is assumed to be on the machine where the script 
is run.  If you can run the script on the router, then the dynamic addresses
are automatically picked up.


BUGS:
   -- need a way to put in real comments... ;# ?


TODO - Improvements I would like to do
   Add named-checkzone
   Adding NS directive.
	have it build name server prolog from the tagged entries.
   Add the tail -f syslog to identify what to add to config.
   Adding auto-dhcp.
	;DNS-Network 192.168.1.0 gateway=1 dhcp (dungeon|guests)=4

Run on other machines.
   This script works a lot better/more automagically if it is run
   on the gateway firewall.  Many use an appliance as a firewall, rather
   than a real debian host.  The script uses /etc/radvd.conf and the interface
   addresses on the gateway hosts to build the zone.  It would be better
   if the script would notice, at a client level, parameters and use
   them, rather than have to run on the gateway.

Build whole named configuration.
   currently builds: named.conf.local, all fwd and rev zones,
   named.conf, named.conf.options, but not: zone.empty,
   db.0, db.127, db.root, etc...
   these files are pretty trivial, did not see the point, but
   if you do not create them, the checkconf remains reliant on /etc/bind.

Improve DHCPv4 automation
   pick an address range automatically.
   automatic guest addresses.
   add email notification for guests.
   add a donjeon mode... (wrong router?, wrong net?) for guests.

Support DHCPv6.
   Currently IPv6 uses SLAC and radvd.  fixed addresses for DHCPv6
   are calculated but not building the config file yet.
   DONE!


Other Stuff:  Firewall Scripts included at no extra cost!

Have a look at the interfaces and rc*firewall scripts.  They give an
idea about how to set that up as well.  In IPv4 you use NAT, but with
IPv6, firewalls do not have to work so hard, you just limit what they
can access with lists.
	
FIXME:
	- sysctl ip_forward ?
	- 

