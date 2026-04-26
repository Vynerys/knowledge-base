How to set up Wireguard on Linux as a server



setup wireguard as server and route all traffic from peers to wireguard server then route to internet



install your Linux distro of choice and update/secure it (refer to Linux best practices)





install wireguard, ufw and iptables if not installed

* sudo apt install wireguard ufw iptables -y





generate a key pair

first private key

* wg genkey | sudo tee /etc/wireguard/private.key
* sudo chmod go= /etc/wireguard/private.key

first command generate a private key and write it to the file private.key in the directory /etc/wireguard/

second command removes all permissions on the file except for root user

output of the first command should be a string of base64 

second public key

* sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key

this command generate a public key from the private key and stores it in the file public.key in the directory /etc/wireguard/

output of the command should also be a string of base64





choosing an ipv4 and ipv6 range of addresses

* ipv4 example 10.1.0.0/24

server will use 10.1.0.1/24 peers will use from 10.1.0.2/24 to 10.1.0.255/24

for ipv6

* date +%s%N
* cat /var/lib/dbus/machine-id

take the output of those two command combine them with date first and machine-id second

them use the command

* printf <the numbers from the date><the numbers for the machine-id> | sha1sum

take the output of the command and use this command

* printf <output of the last command> | cut -c 31-

you should a 10 character string

now add fd at the beginning and separate every 4 character with : and add at the end ::/64

now at the end of ipv6 after the two :: add 1 (::1/64) it will be the ipv6 of the server, peers will use for example ::2/64





( if you want to forward traffic from peers to the internet via the wireguard server then add these lines to /etc/sysctl.conf

* sudo nano /etc/sysctl.conf
* net.ipv4.ip\_forward=1
* net.ipv6.conf.all.forwarding=1

save )





creating wireguard server configuration file

create a file named wg0.conf in /etc/wireguard/

* sudo nano /etc/wireguard/wg0.conf

add these lines in the file

* \[Interface]
* Address = 10.1.0.1/24, ipv6 address
* MTU 1420
* SaveConfig = true
* ListenPort = 51820
* PrivateKey = base64key= (of the server)

replace the values with your chosen values





configuring wireguard's server firewall rules

this is to route the traffic from the virtual interfaces of the vpn to the interfaces connected to the internet

find the server default interface with this command

* ip route list default

remember the interface name for the next section

now edit the server configuration file

* sudo nano /etc/wireguard/wg0.conf

under the last line of the \[Interface] section add these lines

* PostUp = sysctl -w net.ipv4.ip\_forward=1; sysctl -w net.ipv6.conf.all.forwarding=1
* PostUp = ufw route allow in on wg0 out on ens3
* PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
* PostUp = ip6tables -A FORWARD -i %i -j ACCEPT; ip6tables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
* PreDown = sysctl -w net.ipv4.ip\_forward=0; sysctl -w net.ipv6.conf.all.forwarding=0
* PreDown = ufw route delete allow in on wg0 out on ens3
* PreDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
* Predown = ip6tables -D FORWARD -i %i -j ACCEPT; ip6tables -t nat -D POSTROUTING -o ens3 -j MASQUERADE





if firewall (like UFW) add rule to allow wireguard port UDP (51820 if unchanged), add SSH port and other needed ports





starting the wireguard server

use the wg-quick built in script

* sudo systemctl enable wg-quick@wg0.service

wg0 is the name of the configuration file you created at /etc/wireguard/wg0.conf if other name change it in the command

now start the service

* sudo systemctl start wg-quick@wg0.service

to check status

* sudo systemctl status wg-quick@wg0.service





configuring a peer

creating wireguard peer's key pairs (you can create them on the server)

let's first create a folder for the peer

* sudo mkdir /etc/wireguard/peer1

then we create the key pair

* wg genkey | sudo tee /etc/wireguard/peer1/private.key
* sudo chmod go= /etc/wireguard/peer1/private.key

next generate the public key

* sudo cat /etc/wireguard/peer1/private.key | wg pubkey | sudo tee /etc/wireguard/peer1/public.key

create the peer conf file

* sudo nano /etc/wireguard/peer1/wg0.conf

and add these lines in the file

* \[Interfaces]
* PrivateKey = base64key= (of the peer)
* Address = 10.1.0.2/24, ::2/64
* DNS = 1.1.1.1, 1.0.0.1, 213.186.33.99
* MTU = 1420



* \[Peer]
* PublicKey = base64key= (of the server)
* AllowedIPs = 0.0.0.0/0, ::/0
* Endpoint = (IP or Address of the wireguard server) 51.91.11.209:51820





adding peer's public key to the wireguard server

* sudo wg set wg0 peer <peer public key>= allowed-ips 10.1.0.2, ::2



