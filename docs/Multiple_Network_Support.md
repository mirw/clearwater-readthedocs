# Multiple Network Support

## Introduction

As of the Ultima release, Project Clearwater can be configured to provide signalling service (SIP/HTTP/Diameter) on a separate physical (or virtual) network to management services (SNMP/SSH/Provisioning).  This may be used to protect management devices (on a firewalled network) from attack from the outside world (on the signalling network).

## Technical Details

### Network Independence

If traffic separation is configured, the management and signalling networks are accessed by the VM via completely separate (virtual) network interfaces.

The management and signalling networks may be completely independent.

 * There need not be any way to route between them.
 * The IP addresses, subnets and default gateways used on the networks may be the same, may overlap or may be completely distinct.

### Traffic Distinction

The following traffic is carried over the management interface:

 * SSH
 * DNS (for management traffic)
 * SNMP statistics
 * SAS
 * NTP
 * Homestead provisioning
 * Homer provisioning
 * Ellis provisioning API
 * M/Monit

The following traffic is carried over the signalling interface:

 * DNS (for signaling traffic)
 * ENUM
 * SIP between the UE, Bono and Sprout
 * Cx between Homestead and the HSS
 * Rf between Ralf and the CDF
 * Ut between Homer and the UE
 * inter-tier HTTP
 * intra-tier memcached, Cassandra and Chronos flows

Both interfaces respond to ICMP pings which can be used by external systems to perform health-checking if desired.

## Configuration

### Pre-requisites

The (possibly virtual) machines that are running your Project Clearwater services will need to be set up with a second network interface (referred to as `eth1` in the following).

You will need to determine the IP configuration for your second network, including IP address, subnet, default gateway, DNS server.  DHCP is supported on this interface if required but you should ensure that the DHCP lease time is sufficiently high to ensure that the local IP address does not change regularly.

### Preparing the network

Due to the varied complexities of IP networking, it would be impractical to attempt to automate configuring the Linux-level view of the various networks.  Network namespaces are created and managed using the `ip netns` tool, which is a standard part of Ubuntu 12.04.

The following example commands (when run as root) create a network namespace, move `eth1` into it, configure a static IP address and configure routing.

    ip netns add signalling
    ip link set eth1 netns signalling
    ip netns exec signalling ifconfig lo up
    ip netns exec signalling ifconfig eth1 1.2.3.4/16 up
    ip netns exec signalling route add default gateway 1.2.0.1 dev eth1

Obviously you should make any appropriate changes to the above to correctly configure your chosen signalling network.  These changes are **not** persisted across reboots on Linux and you should ensure that these are run on boot before the `clearwater-infrastructure` service is run.  A sensible place to configure this would be in `/etc/rc.local`.

Finally, you should create `/etc/netns/signalling/resolv.conf` configuring the DNS server you'd like to use on the signalling network.  The format of this file is documented at <http://manpages.ubuntu.com/manpages/precise/man5/resolv.conf.5.html> but a simple example file might just contain the following.

    nameserver <DNS IP address>

### Project Clearwater Configuration

Now that the signalling namespace is configured and has networking set up, it's time to apply it to Project Clearwater.  To do this, add the following two lines to `/etc/clearwater/config`:

    signaling_namespace=<namespace name>
    signaling_dns_server=<DNS IP address>

And change the `local_ip` line to the IP address on the signalling network.

If you've not yet installed the Project Clearwater services, do so now and the namespaces will be automatically used.

If you've already installed the Project Clearwater services, run `sudo service clearwater-infrastructure restart` and restart all the Clearwater-related services running on each machine (you can see what these are with `sudo monit summary`).

## Diagnostic Changes

All the built-in Clearwater diagnostics will automatically take note of network namespaces, but if you are running diagnostics yourself (e.g. following instructions from <https://github.com/Metaswitch/clearwater-docs/wiki/Troubleshooting-and-Recovery>) you may need to prefix your commands with `ip netns exec <namespace>` to run them in the signalling namespace.  The following tools will need this prefix:

 * `cqlsh -3` - For viewing Cassandra databases
 * `nodetool` - For viewing Cassandra status
 * `telnet` - For inspecting Memcached stores