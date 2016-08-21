---
layout: post
title: "Creating BOINC clusters"
---


# Overview

Assume you have multiple computers idle and want to run [BOINC projects](http://boinc.berkeley.edu/index.php) on them. Do you install each OS instance separately on each machine's hard drive, and then monitor the computers individually? What about updates or hardware replacements? It quickly gets very time-consuming to manage multiple computers.

There is another way however: you can setup one Linux machine as a PXE boot server, and then have all the other machines boot from it and then mount their root filesystem from a NFS share.

* This will become a table of contents
{:toc}


# Requirements

- **Debian/Ubuntu Linux** installed. This guide assumes using Debian Wheezy, but others should be similar.
- Network cards that support PXE booting, preferably gigabit speed. Usually all of them support PXE, but may need to be activated from BIOS (look for **LAN Boot ROM** or similar). In this setup, the server needs 2 NICs.
- Hard drive only in the PXE boot server
- Moderate Linux administration skills

# Preparations

The file server does not need much computing power, only a couple GB for file caching, a fast disk, and a fast network adapter. It does not necessarily need to run boinc-client so yo can just skip installing that package. All commands assume you are running as root.

Here's an outline for the network setup:

	Server -- eth1 / 12.34.56.78 --> Internet (NAT masquerade the cluster)
	      `-- eth0 / 10.0.0.1 -----> Cluster LAN
	
	Node-1 -- eth0 / 10.0.0.11 -----> Cluster LAN
	Node-2 -- eth0 / 10.0.0.12 -----> Cluster LAN
	               - static IP assigned by server's DHCP
	               - DNS server also 10.0.0.1


Install requirements on server:

	apt-get install nfs-kernel-server tftpd-hpa isc-dhcp-client isc-dhcp-server debootstrap syslinux boinc-manager netplug dnsmasq

Some optional monitoring utilities:

	sinfo syslog-ng lm-sensors

# Set up the DHCP server

DHCP will assign nodes static IPs by their ethernet MACs.

**/etc/dhcp/dhcpd.conf**

	option domain-name-servers 10.0.0.1;
	option routers 10.0.0.1;
	next-server 10.0.0.1;
	use-host-decl-names on;

	default-lease-time 3600;
	max-lease-time 7200;

	allow booting;
	allow bootp;

	authoritative;
	log-facility local7;

	# If an unknown computer connects, serve IPs 10.0.0.20->
	subnet 10.0.0.0 netmask 255.255.255.0 {
		range 10.0.0.20 10.0.0.254;
		filename "/pxelinux.0";
	}

	host node-1 {
		hardware ethernet 11:22:33:44:55:66;
		fixed-address 10.0.0.11;
	}
	host node-2 {
		hardware ethernet 12:12:12:12:12:12;
		fixed-address 10.0.0.12;
	}

# Set up the DNS server

**/etc/dnsmasq.conf**

	interface=eth0
	no-dhcp-interface=eth0


# Configure the network

**/etc/hosts**

	10.0.0.1  server
	10.0.0.11 node-1
	10.0.0.12 node-2


**/etc/network/interfaces**

	auto lo
	iface lo inet loopback
	
	auto eth0 eth1
	allow-hotplug eth0 eth1

	iface eth1 inet dhcp

	iface eth0 inet static
		address 10.0.0.1
		netmask 255.255.255.0

# Configure forwarding and firewalling

**/etc/sysctl.conf** (add/uncomment the following lines)

	net.ipv4.ip_forward = 1
	net.ipv4.conf.default.rp_filter = 1

Now let's setup iptables:

	iptables -F
	iptables -t nat -F

	export LAN=eth0
	export WAN=eth1

	iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
	iptables -A INPUT -m state --state INVALID -j DROP
	iptables -A INPUT -i lo -j ACCEPT
	iptables -A INPUT -i ${LAN} -j ACCEPT
	iptables -A INPUT --protocol icmp -j ACCEPT
	iptables -A INPUT --protocol tcp --dport 22 -j LOG --log-prefix "SSH "
	iptables -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT

	iptables -I FORWARD -i ${LAN} -d 10.0.0.0/24 -j DROP
	iptables -A FORWARD -i ${LAN} -s 10.0.0.0/24 -j ACCEPT
	iptables -A FORWARD -i ${WAN} -d 10.0.0.0/24 -j ACCEPT
	iptables -t nat -A POSTROUTING -o ${WAN} -j MASQUERADE

	iptables -P INPUT DROP
	iptables -P FORWARD DROP

This means iptables will accept everything from the LAN interface, while rejecting everything except SSH from the WAN interface. LAN connections to WAN will also be masqueraded.

To restore settings automatically after reboot:

	iptables-save > /etc/tables.conf
	echo '#!/bin/sh' > /etc/network/if-up.d/iptables
	echo "iptables-restore < /etc/tables.conf" >> /etc/network/if-up.d/iptables
	chmod +x /etc/network/if-up.d/iptables
	chmod 750 /etc/tables.conf


# Creating the NFS system

It is recommended to put all shares on `/srv`. For example:

	cd /srv
	mkdir mainfs node-1 node-2 tftpboot

All nodes mount `/srv/mainfs` as their root filesystem and then `/srv/node-#` on `/var/lib/boinc-client` so that each instance has their own BOINC data directory.

Now we need to bootstrap the installation:

	debootstrap wheezy /srv/mainfs

After that, `mainfs` should have a "barebones" Debian installation.


# Configure your installation

**/srv/mainfs/etc/network/interfaces**

	auto lo
	iface lo inet loopback
	iface eth0 inet manual

You could change the password before booting with the following:

	chroot /srv/mainfs
	passwd

While chrooted, install the node's requirements:

	apt-get update
	apt-get install libc6-i386 nfs-common boinc-client sinfo syslog-ng lm-sensors

Then close the terminal.


# Configuring TFTPD

**/etc/default/tftpd-hpa**

	TFTP_USERNAME="tftp"
	TFTP_DIRECTORY="/srv/tftpboot"
	TFTP_ADDRESS="10.0.0.1:69"
	TFTP_OPTIONS="--secure --listen -vvv"


# PXELinux.cfg

Assuming we use the stock kernel version `3.2.0-4-amd64`:

	cp /boot/*3.2.0-4-amd64 /srv/tftpboot/
	mkdir /srv/tftpboot/pxelinux.cfg
	cp /usr/lib/syslinux/pxelinux.0 /srv/tftpboot

**/srv/tftpboot/pxelinux.cfg/default**

	DEFAULT linux
	LABEL linux
	KERNEL vmlinuz-3.2.0-4-amd64
	APPEND root=/dev/nfs initrd=initrd.img-3.2.0-4-amd64 nfsroot=10.0.0.1:/srv/mainfs ip=dhcp rw


# Configuring the NFS mounts

**/etc/exports**

	/srv/mainfs 10.0.0.0/24(rw,async,no_root_squash,no_subtree_check)
	/srv/node-1 10.0.0.11(rw,async,no_root_squash,no_subtree_check)
	/srv/node-2 10.0.0.12(rw,async,no_root_squash,no_subtree_check)

Now we need to also specify the mounts on **client-side**.

**/srv/mainfs/etc/fstab**

	proc            /proc           proc    defaults        0       0
	/dev/nfs       /               nfs    defaults          1       1
	none            /tmp            tmpfs   defaults        0       0
	none            /var/run        tmpfs   defaults        0       0
	none            /var/lock       tmpfs   defaults        0       0
	none            /var/tmp        tmpfs   defaults        0       0
	/dev/hda        /media/cdrom0   udf,iso9660 user,noauto 0       0

We can't include conditional mounts based on hostname in `fstab`, so we need to create a script that runs on boot to mount something on client's `/var/lib/boinc-client`.

**/srv/mainfs/etc/init.d/mount-boinc-data**

	#! /bin/sh
	## BEGIN INIT INFO
	# Provides:          mount-boinc-data
	# Required-Start:    mountnfs
	# Required-Stop:
	# Should-Start:
	# Default-Start:     2 3 4 5
	# Default-Stop:      0 1 6
	# Short-Description: Wait for network data filesystems to be mounted
	## END INIT INFO

	. /lib/init/vars.sh
	. /lib/lsb/init-functions

	mount_nfs_data() {
		[ -f /sbin/mount.nfs ] || return
		[ -f /bin/hostname ] || return

		# Wait for each path, the timeout is for all of them as that's
		# really the maximum time we have to wait anyway
		TIMEOUT=900
		MOUNTPT="/var/lib/boinc-client"
		NFSDIR="10.0.0.1:/srv/`/bin/hostname`"

		log_action_begin_msg "Waiting for $NFSDIR"
		mount.nfs $NFSDIR $MOUNTPT
		sleep 0.1

		TIMEOUT=$(( $TIMEOUT - 1 ))
		if [ $TIMEOUT -le 0 ]; then
			log_action_end_msg 1
			break
		fi

		if [ $TIMEOUT -gt 0 ]; then
			log_action_end_msg 0
		fi
	}
	case "$1" in
		start)
		mount_nfs_data
			;;
		restart|reload|force-reload)
			echo "Error: argument '$1' not supported" >&2
			exit 3
			;;
		stop)
			;;
		*)
			echo "Usage: $0 start|stop" >&2
			exit 3
			;;
	esac
	: exit 0

The crucial bit here is the execution of `/bin/hostname` which helps mount a share named `node-1` etc.

However, boinc-client can't start before it's data directory is mounted, so we need to modify `/srv/mainfs/etc/init.d/boinc-client`. Find a line starting with `# Required-Start:` and change it to:

	# Required-Start:    $local_fs $remote_fs mount-boinc-data

Set permissions:

	chmod a+x /srv/mainfs/etc/init.d/*


# Configure network syslogging

To monitor multiple machines it is recommended to send the syslogs to a centralized log server, in this case, the NFS server.

**/etc/syslog-ng/syslog-ng.conf** (change existing or add the lines)

	options { flush_lines(0); use_dns(no); use_fqdn(no);
			owner("root"); group("adm"); perm(0640); stats_freq(0);
			bad_hostname("^gconfd$");
			check_hostname(yes);
			keep_hostname(yes);
			chain_hostnames(no);
	};

	source s_net { tcp(ip(10.0.0.1) port(1000)); };
	destination remote { file("/var/log/remote/$FULLHOST"); file("/dev/tty10"); };
	log { source(s_net); destination(remote); };

**/srv/mainfs/etc/syslog-ng/syslog-ng.conf**

	destination d_console_all { file("/dev/tty10"); };
	destination d_net { tcp("10.0.0.1" port(1000) log_fifo_size(1000)); };

Then remove **all** destination lines having /var/log like this:

	destination d_something { file("/var/log/something"); };

Now you can view messages from the whole cluster on tty10 or browse logs on /var/log/remote.


# Configure BOINC management

**/srv/mainfs/etc/boinc-client/remote_hosts.cfg**

	10.0.0.1

By default, BOINC will put configuration files in their data directory. Let's make sure all nodes use the same configuration by symlinking them to the same files.

	mkdir /srv/node-default
	cd /srv/node-default
	ln -s /etc/ssl/certs/ca-certificates.crt ca-bundle.crt
	ln -s /etc/boinc-client/cc_config.xml cc_config.xml
	ln -s /etc/boinc-client/global_prefs_override.xml global_prefs_override.xml
	ln -s /etc/boinc-client/gui_rpc_auth.cfg gui_rpc_auth.cfg
	ln -s /etc/boinc-client/remote_hosts.cfg remote_hosts.cfg

Copy the template directory to any newly created nodes:

	cp -f /srv/node-default/* /srv/node-1/

Etc.

# Configure SSH key login

It is recommended to setup RSA key login to the cluster nodes.

	ssh-keygen -t rsa
	mkdir -m 700 -p /srv/mainfs/root/.ssh
	cat ~/.ssh/id_rsa.pub > /srv/mainfs/root/.ssh/authorized_keys

To disable remote password login, just set `PasswordAuthentication no` in `/srv/mainfs/etc/ssh/sshd_config` or `/etc/ssh/sshd_config`.

# Wrapping up

To finish, reboot the server because configuration has been changed. Then attempt to PXE boot a node, and watch tty10 for any messages if problems arise. If all is okay, you should be able to SSH to the node.

In the BOINC manager, connect to the node IP such as 10.0.0.11 and then attempt to attach to a project. Make sure the individual NFS mounts are working before starting multiple instances!

If you installed sinfo on each machine, now it's time to monitor the cluster (e.g. stare aimlessly in awe at the status screen):

	sinfo -snd2 -t3 -c full


# Other resources

- [Diskless Ubuntu Howto](https://help.ubuntu.com/community/DisklessUbuntuHowto)
- [Creating a diskless cluster](http://web.archive.org/web/20120419095619/http://www.boinc-wiki.info/Creating_a_diskless_cluster)
- [Debian/Ubuntu Diskless Folding HOWTO](http://web.archive.org/web/20120122151945/http://reilly.homeip.net/folding/linux.html)



