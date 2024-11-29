######################################################
------------------------------------------------------
Author: Haroun Kassouri  
Date: 28/11/2024  

Systems Integration Asignment 2 Howto Document
------------------------------------------------------
######################################################

------------------------------------------------------
Introduction
------------------------------------------------------
DNS, DHCP, NFS, FTP, and Router Setup Guide  

This guide provides step-by-step instructions for configuring DNS, DHCP, NFS, FTP servers, and a router on a Linux system. Each step includes commands, their explanations, and exact file locations.
This document assumes basic knowledge of Linux and networking concepts.

------------------------------------------------------
Table of Contents
------------------------------------------------------
1. Setting Up a DNS Server  
2. Setting Up a DHCP Server  
3. Setting Up an NFS Server  
4. Setting Up an FTP Server  
5. Setting Up a Router  

------------------------------------------------------
Part 1. Setting Up a DNS Server
------------------------------------------------------
To set up a DNS server for the domain `example.lan`, follow these steps:

1. **Install BIND9**  
Run the following commands to update the package list and install the BIND9 DNS server:
sudo apt update
sudo apt install bind9


2. **Configure the Forward Lookup Zone**  
Edit the BIND configuration file to define the forward lookup zone:
sudo nano /etc/bind/named.conf.local

Add the following block od code:
zone "example.lan" { 
type master; file "/etc/bind/db.example.lan"; 
};

This code specifies the domain 'example.lan' that the DNS server will manage and the location of its config file.


3. **Create the Forward Zone File**  
Create a new zone file for 'example.lan':
sudo nano /etc/bind/db.example.lan

Edit the file with this code hjere:
$TTL 604800 @ IN SOA ns.example.lan. admin.example.lan. ( 
20241128 ; Serial 604800 ; Refresh 86400 ; Retry 2419200 ; Expire 604800 
) ; Negative Cache TTL @ IN NS ns.example.lan. ns IN A 192.168.1.1 www IN A 192.168.1.2

This file specifies the DNS records for the domain, including the IP for 'example.lan'.


4. **Configure the Reverse Lookup Zone**  
Edit the configuration file to add a reverse lookup zone for mapping IP addresses back to domain names:
sudo nano /etc/bind/named.conf.local

Add:
zone "1.168.192.in-addr.arpa" { 
type master; file "/etc/bind/db.192.168.1"; 
};


5. **Create the Reverse Zone File**
sudo nano /etc/bind/db.192.168.1

Place block of line in cod:
$TTL 604800 @ IN SOA ns.example.lan. admin.example.lan. ( 
20241128 ; Serial 604800 ; Refresh 86400 ; Retry 2419200 ; Expire 604800 
) ; Negative Cache TTL @ IN NS ns.example.lan. 1 IN PTR example.lan.

This file maps the IP '192.168.1.1' back to 'example.lan'.


6. **Restart and Test DNS**  
sudo systemctl restart bind9

Test forward and reverse lookups:
Forward: dig @192.168.1.1 example.lan 
Reverse: dig @192.168.1.1 -x 192.168.1.1


------------------------------------------------------
Part 2: Setting Up a DHCP Server
------------------------------------------------------
This section explains how to configure a DHCP server to allocate IP addresses dynamically.

### Server-Side Configuration

1. **Install the DHCP Server**  
sudo apt update
sudo apt install isc-dhcp-server

2. **Configure the dhcp server**  
sudo nano /etc/dhcp/dhcpd.conf
Add the following block:

subnet 192.168.1.0 netmask 255.255.255.0 { 
    range 192.168.1.150 192.168.1.200;
    option routers 192.168.1.1;
    option domain-name-servers 192.168.1.1;
    default-lease-time 600;
    max-lease-time 7200;
}

Specififying range of IP addresses to allocate while binding dhcp server

3. **Edit the interface configuration file**  
sudo nano /etc/default/isc-dhcp-server

Update the line to the linked network that will connect between the vm and vm clone which will be used as the client:
INTERFACESv4="enp0s8"

4. **Restart and Test the DHCP Server**  
sudo systemctl restart isc-dhcp-server

Check active leases:
sudo dhcp-lease-list

This should show the client side with an ip address of the range 150 - 200 


### Client-Side Configuration

5. **Set Up the Client Machine (VM Clone)**
Ensure the client machine is connected to the same network as the server (Via the same internal network on adaptor 2).

Configure the client machine to use DHCP:
sudo nano /etc/netplan/01-netcfg.yaml

Add the following block of code:
network:
    version: 2
    ethernets:
        enp0s8:
            dhcp4: true

Apply the configuration:
sudo netplan apply

This ensures the client will request an IP address from the DHCP server.

Reboot the client machine to make sure it understands this:
sudo reboot

6. **Verify the IP Address on the Client**
After the client machine reboots, check the IP address:
ip addr

The client should have an IP address within the range 192.168.1.150 - 192.168.1.200.

Test connectivity by pinging the DHCP server:
e.g. ping 192.168.1.1

7. **Verify Leases on the Server**
On the server, check the list of active leases:
sudo dhcp-lease-list

This command displays the details of IP addresses that the DHCP server has allocated to clients.

8. **Potential Errors**  
Ensure the zone file permissions are correct as this is a very common issue that occured for me. This is a useful line.
sudo chown bind:bind /etc/bind/db.example.lan

Connectivity Issues: Ensure the server and client are on the same subnet and there are no firewall rules blocking communication.


------------------------------------------------------
Part3. Setting Up an NFS Server
------------------------------------------------------

### Server-Side Configuration

1. **Install the NFS Server**  
sudo apt update
sudo apt install nfs-kernel-server

2. **Create and Configure Permisions for the Shared Directory**  
Create a directory to be shared and set appropriate permissions to be able to use that directorty:
mkdir -p /home/<username>/shared 
sudo chown <username>:<username> /home/<username>/shared    
sudo chmod 777 /home/<username>/shared

3. **Configure NFS Exports**  
Open the exports file to define which directories will be shared:
sudo nano /etc/exports

Add the following line:
/home/<username>/shared <server IP address>(rw,sync,no_subtree_check)

4. **Restart and Test the NFS Server**  
Apply the configuration and restart the service:
sudo exportfs -a 
sudo systemctl restart nfs-kernel-server 


### Client-Side Configuration

5. **Install the NFS Client**  
Install the NFS client tools on the client machine:
sudo apt update 
sudo apt install nfs-common


6. **Mount the Shared Directory**  
Create a directory on the client and mount the shared directory:
mkdir -p /mnt/shared 
sudo mount <server IP address>:/home/<username>/shared /mnt/shared

7. **Test the Connection**  
Navigate to /mnt/shared and verify access:
cd /mnt/shared ls

If the folders are the same even if changes are made than it is correct. Try created a few folders and files to test it out

------------------------------------------------------
Part4. Setting Up an FTP Server
------------------------------------------------------

### Server-Side Configuration

1. **Install vsftpd**  
sudo apt update 
sudo apt install vsftpd

2. **Configure the FTP Server**  
Edit the configuration file:
sudo nano /etc/vsftpd.conf

Make sure the the following are in the file:
anonymous_enable=YES
pasv_enable=YES 
pasv_min_port=10000 
pasv_max_port=10010

These allow anonymous FTP acces and enable paassive mode for FTP

3. **Restart and Test FTP**  
sudo systemctl restart vsftpd

Test FTP access from the client by pinging to server ip address
ftp <server IP address>


### Client-Side Configuration

4. **Install an FTP Client**  
sudo apt update 
sudo apt install ftp


5. **Connect to the FTP Server**  
ftp <server IP address>


6. **Test File Transfer**  
Once connected, test uploading or downloading files:
get <filename> 

Can also test using: 
wget ftp://server IP address/my_file.txt  (make sure to leave ftp, test this in normal command line)


------------------------------------------------------
Part 5. Setting Up a Router
------------------------------------------------------
This will show how arouter will allow a client machine to access the internet through the server.

### Server-Side Configuration

1. **Enable Packet Forwarding on the Server**  
Open the system configuration file:
sudo nano /etc/sysctl.conf

Uncomment the following line:
net.ipv4.ip_forward=1

This allows the server to forward network packets.

2. **Configure IP Tables for Packet Forwarding**  
Allow forwarding from the server’s interface enp0s8 (connected to client) to enp0s3 (connected to Internet):
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT

Allow forwarding in the reverse direction (enp0s3 to enp0s8):
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT

Enable NAT (Network Address Translation) on the outgoing interface (enp0s3):
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE


3. **Save IP Tables Rules**

Install the package to persist iptables rules:
sudo apt install iptables-persistent

Save the current rules to make them permanent:
sudo bash -c "iptables-save > /etc/iptables/rules.v4"

This ensures your NAT and forwarding rules persist after a reboot.

4. **Configure Firewall to Allow NAT Packets**
Open the UFW configuration file:
sudo nano /etc/default/ufw

Change the following line:
DEFAULT_FORWARD_POLICY="DROP"
to:
DEFAULT_FORWARD_POLICY="ACCEPT"

Open the UFW sysctl configuration file:
sudo nano /etc/ufw/sysctl.conf

Add the following line:
net/ipv4/ip_forward=1

Restart UFW to apply the changes:
sudo ufw disable
sudo ufw enable

Save the changes:
sudo netfilter-persistent save

### Client-Side Configuration
5. **Configure the Client**
Open the Netplan configuration file on the client:
sudo nano /etc/netplan/99_config.yaml

Replace the file content with:
network:
    version: 2
    renderer: networkd
    ethernets:
        enp0s8:
            dhcp4: true
            routes:
                - to: default
                  via: 192.168.40.10
            nameservers:
                addresses: [192.168.40.10]

Save and apply the changes:
sudo netplan apply

6. **Remove Client’s Direct Internet Access**

Shut down the client machine.

Open VirtualBox settings for the client VM:
Navigate to Settings > Network > Adapter 1. 
Uncheck the Enable Network Adapter option.

Disabling Adapter 1 forces the client to route traffic through the server.

Reboot the client.

7. **Test Connectivity**

On the client machine, test connectivity to the Internet:
ping google.com

If the client can successfully access external websites, the server has been set up correctly as a router.
