## Isolation

## Chroot
Link: <https://en.wikipedia.org/wiki/Chroot>

**Chroot** is an operation in Unix OSes that changes the apparent root directory for the current running process and children. The program in a chroot environment cannot name and therefore normally access files outside of the designated directory.

Chroot implements the concept of jailbreak. A chroot environment can be used to create and host a separate __virtualized copy__ of the software system.


### Limitations

Chroot is not enough to contain a process with **root privileges** or to defend against intentional tampering by privileged users.

To avoid jailbreaking with a second chroot, chrooted programs should relinquishing root privileges.

In addition, upon startup most programs expect to find scratch spaces, device nodes and shared libraries at certain preset locations. Thus for a chrooted program to successful start, the directory must be populated with a minimum set of these files.

## LXC - Linux Containers

LXC has a stable C API with bindings for other various programming languages.

It is an OS  level virtualization method for running multiple isolated containers on a host using a single Linux kernel. It uses a virtual environment with its own process and network space instead of creating a full-fledged VM.

## iptables

Iptables is a CLI utility for configuring kernel-level firewall. The iptables are used to inspect, modify, forward, redirect, and/or drop IP packets. 

It is a front-end to the kernel-level netfilter hooks that can manipulate the Linux network stack.

The tables are made of a set of predefined *chains*, and the chains contain rules which each packet is compared against in order. Each rule makes up of a predicate of potential matches and a corresponding action called **target** that is executed if the predicate is true, like an accept or drop. Within a chain there can only at most  one match.

## Chains

There are three chains by default:
* Input: Handle all packets addressed to server
* Output: Contains rules for traffic created by server
* Forward: Deal with traffic destined for other server.

Each chain contains zero or more rules, and has a default **policy**. The policy determines what happens when a packet does not match any rule in chain.

### Tables

iptables contains five tables:

* `raw` is used only for configuring packets so that they are exempt from connection tracking.
* `filter` is the default table, and is where all the actions typically associated with a firewall take place.
* `nat` is used for network address translation (e.g. port forwarding).
* `mangle` is used for specialized packet alterations.
* `security` is used for Mandatory Access Control networking rules (e.g. SELinux)

