######################################################
------------------------------------------------------
Author: Haroun Kassouri  
Date: 28/11/2024  
------------------------------------------------------
######################################################

------------------------------------------------------
Introduction
------------------------------------------------------
# DNS, DHCP, NFS, FTP, and Router Setup Guide  

This guide provides step-by-step instructions for configuring DNS, DHCP, NFS, FTP servers, and a router on a Linux system. Each step includes commands, their explanations, and exact file locations, ensuring clarity for anyone following the setup.

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
Part1. Setting Up a DNS Server
------------------------------------------------------
To set up a DNS server for the domain `example.lan`, follow these steps:

1. Install BIND9:  
Run the following commands to update the package list and install the BIND9 DNS server:
sudo apt update
sudo apt install bind9


2. Configure the Forward Lookup Zone:  
Edit the BIND configuration file to define the forward lookup zone:
sudo nano /etc/bind/named.conf.local

Add the following block:
zone "example.lan" { 
type master; file "/etc/bind/db.example.lan"; 
};

This block specifies the domain 'example.lan' that the DNS server will manage and the location of its config file.


3. Create the Forward Zone File:  
Create a new zone file for 'example.lan':
sudo nano /etc/bind/db.example.lan

Edit the file with this code hjere:
$TTL 604800 @ IN SOA ns.example.lan. admin.example.lan. ( 
20241128 ; Serial 604800 ; Refresh 86400 ; Retry 2419200 ; Expire 604800 
) ; Negative Cache TTL @ IN NS ns.example.lan. ns IN A 192.168.1.1 www IN A 192.168.1.2

This file specifies the DNS records for the domain, including the name server (`ns.example.lan`) and IP for `example.lan`.


4. Configure the Reverse Lookup Zone:  
Edit the configuration file to add a reverse lookup zone for mapping IP addresses back to domain names:
sudo nano /etc/bind/named.conf.local

Add:
zone "1.168.192.in-addr.arpa" { 
type master; file "/etc/bind/db.192.168.1"; 
};


5. Create the Reverse Zone File:  
Create file:
sudo nano /etc/bind/db.192.168.1

Place block of line in cod:
$TTL 604800 @ IN SOA ns.example.lan. admin.example.lan. ( 20241128 ; Serial 604800 ; Refresh 86400 ; Retry 2419200 ; Expire 604800 ) ; Negative Cache TTL @ IN NS ns.example.lan. 1 IN PTR example.lan.

This file maps the IP `192.168.1.1` back to `example.lan`.


6. Restart and Test DNS:  
Validate the zone files and restart the DNS server:
sudo named-checkzone example.lan /etc/bind/db.example.lan
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.192.168.1 
sudo systemctl restart bind9

Test forward and reverse lookups:
Forward: dig @192.168.1.1 example.lan 
Reverse: dig @192.168.1.1 -x 192.168.1.1


------------------------------------------------------
Part2. Setting Up a DHCP Server
------------------------------------------------------
To set up a DHCP server for automatic IP allocation:

1. Install the DHCP Server:  
sudo apt update 
sudo apt install isc-dhcp-server

2. Configure the DHCP Server:  
Edit the main configuration file:
sudo nano /etc/dhcp/dhcpd.conf

Add:
subnet 192.168.1.0 netmask 255.255.255.0 { 
range 192.168.1.150 192.168.1.200; option routers 192.168.1.1; option domain-name-servers 192.168.1.1; default-lease-time 600; max-lease-time 7200; 
}

This configuration sets the IP range and network parameters.

3. Bind the DHCP Server to the Correct Interface:  
sudo nano /etc/default/isc-dhcp-server

Edit:
INTERFACESv4="enp0s8"


4. Restart and Test the DHCP Server:  
sudo systemctl restart isc-dhcp-server 
sudo tail -f /var/log/syslog 
sudo dhcp-lease-list


------------------------------------------------------
Setting Up an NFS Server
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

1. **Install the NFS Client**  
Install the NFS client tools on the client machine:
sudo apt update 
sudo apt install nfs-common


2. **Mount the Shared Directory**  
Create a directory on the client and mount the shared directory:
mkdir -p /mnt/shared 
sudo mount <server IP address>:/home/<username>/shared /mnt/shared

3. **Test the Connection**  
Navigate to /mnt/shared and verify access:
cd /mnt/shared ls



------------------------------------------------------
Setting Up an FTP Server
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

These allow anonymous FTP acces and enable paaive mode for FTP

3. **Restart and Test FTP**  
sudo systemctl restart vsftpd

Test FTP access from the client by pinging to server ip address
ftp <server IP address>


### Client-Side Configuration

1. **Install an FTP Client**  
sudo apt update 
sudo apt install ftp


2. **Connect to the FTP Server**  
ftp <server IP address>


3. **Test File Transfer**  
Once connected, test uploading or downloading files:
get <filename> put <filename>



------------------------------------------------------
Setting Up a Router
------------------------------------------------------
A router allows the client machine to access the internet through the server.

### Server-Side Configuration

1. **Enable IP Forwarding**  
Open the system configuration file:
sudo nano /etc/sysctl.conf

Uncomment the following line:
net.ipv4.ip_forward=1

- This allows the server to forward network packets.

2. **Configure NAT and Packet Forwarding**  
Add rules to enable NAT (Network Address Translation) and packet forwarding:
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

- `FORWARD`: Forwards packets between interfaces.
- `MASQUERADE`: Hides client IP addresses behind the server's IP for internet access.

3. **Make Changes Persistent**  
Save the iptables rules:
sudo apt install iptables-persistent sudo iptables-save > /etc/iptables/rules.v4

- Ensures NAT and forwarding rules persist after a reboot.

4. **Test Internet Connectivity**  
From the client machine, test the connection:
ping google.com


### Client-Side Configuration

1. **Modify Network Settings**  
Configure the client to route traffic through the server. Edit the network configuration file:
sudo nano /etc/netplan/00-installer-config.yaml

Set:
routes:

to: default via: <server IP address> nameservers: addresses: [8.8.8.8, 8.8.4.4]
- Replace `<server IP address>` with the server’s IP.

2. **Apply the Configuration**  
Apply the changes:
sudo netplan apply


3. **Test Internet Access**  
Verify internet access by pinging external websites:
ping google.com
