# WireGuard-VPN

Virtual Private Networks (VPNs) are essential for maintaining privacy and secure access to your network. WireGuard is a modern VPN that is fast, simple, and secure. In this guide, I’ll show you how to set up a WireGuard VPN server on AWS, giving you the ability to securely connect to your network from anywhere in the world.

# Prerequisites
Before you begin, ensure you have the following:

An AWS account
Basic knowledge of AWS EC2
SSH client to connect to your server
Basic Linux command-line knowledge
Basic Networking Knowledge

# WireGuard — VPN

WireGuard is a modern, open-source Virtual Private Network (VPN) protocol that aims to be simpler, faster, and more secure than traditional VPN solutions like IPsec and OpenVPN. Developed by Jason A. Donenfeld, WireGuard uses state-of-the-art cryptography to create secure point-to-point connections between devices. It is designed to be lightweight, with a minimal codebase, making it easier to audit for security vulnerabilities and faster in performance.

# Key Features of WireGuard:

Simplicity: With fewer lines of code, it is easier to implement and maintain.
Speed: Its efficiency makes it suitable for high-performance applications.
Security: Utilizes modern cryptographic techniques, providing robust encryption.
Cross-Platform: Available for multiple platforms, including Linux, Windows, macOS, iOS, and Android.

# WireGuard VPN Server in AWS

Deploying a WireGuard VPN server in Amazon Web Services (AWS) provides a secure and scalable way to connect to a private network or the internet via an encrypted tunnel. By configuring a WireGuard VPN server in AWS, you can ensure that all your internet traffic is encrypted and routed through a secure server, which is particularly useful when accessing resources remotely or when using public Wi-Fi.

# Server Setup:
OS: Ubuntu 20.04 /22.04

# Launch an EC2 Instance on AWS

Log in to your AWS Management Console.
Navigate to the EC2 Dashboard and click “Launch Instance.”
Choose an Amazon Machine Image (AMI) — I recommend Ubuntu 22.04 LTS.
Select an instance type. For personal use, a t2.micro instance (free tier eligible) is sufficient.
Configure your instance details and storage.
Under “Security Group,” add rules to allow traffic for SSH (port 22) and WireGuard (port 51820/UDP).
Review and launch your instance, and make sure to download the key pair.

# Install Wireguard on AWS EC2 Instance:
```
sudo apt update 
sudo apt install wireguard
```
Now, generate a private and public keypair for the server using “wg genkey” and “wg pubkey” commands.

Private Key:
```
wg genkey | sudo tee /etc/wireguard/private.key 
sudo chmod go= /etc/wireguard/private.key ( # Remove access to non-root users)
```
Public Key:
```
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
```

# Create Wireguard server configuration:

Choose the IP range for the VPN tunnel interface from the following private IP pools.

10.0.0.0 to 10.255.255.255
172.16.0.0 to 172.31.255.255
192.168.0.0 to 192.168.255.255
Example: 10.0.0.1/24

Create a new configuration file:
```
sudo nano /etc/wireguard/wg0.conf
```
Add the following lines to the file, substituting your private key in place of the highlighted “Server private_key_goes_here” value, and the IP address(es) on the Address line. You can also change the ListenPort line if you would like WireGuard to be available on a different port:
```
[Interface] 
PrivateKey = server private_key_goes_here 
Address = 10.0.0.1/24 
ListenPort = 51820 
SaveConfig = true
```
The SaveConfig line ensures that when a WireGuard interface is shutdown, any changes will get saved to the configuration file.

# Network Configuration:

If you are using WireGuard to connect a peer to the WireGuard Server to access services only, then you do not need to complete this section. If you would like to route your WireGuard Peer’s Internet traffic through the WireGuard Server then you will need to configure IP forwarding by following this section.

To configure forwarding, open the /etc/sysctl.conf file using nano or your preferred editor:
```
sudo nano /etc/sysctl.conf
```
Set the IPv4 and IPv6 ip forward enable
```
net.ipv4.ip_forward=1 
net.ipv6.conf.all.forwarding=1
```
To load the configuration, use the command
```
sudo sysctl -p
```
Configuring Wireguard server’s firewall:
In this section, you will edit the WireGuard Server’s configuration to add firewall rules that will ensure traffic to and from the server and clients is routed correctly. As with the previous section, skip this step if you are only using your WireGuard VPN for a machine-to-machine connection to access resources that are restricted to your VPN.

To allow WireGuard VPN traffic through the Server’s firewall, you’ll need to enable masquerading, which is an iptables concept that provides on-the-fly dynamic network address translation (NAT) to correctly route client connections.

First, find the public network interface of your WireGuard Server using the ip route sub-command:
```
ip route list default
```
The public interface is the string found within this command’s output that follows the word “dev”. For example, this result shows the interface named eth0, which is highlighted below:

Output:

default via 203.0.113.1 dev eth0 proto static

Note your network device’s name (eth0 in this case) since you will add it to the iptables rules in the next step.

To add firewall rules to your WireGuard Server, open the /etc/wireguard/wg0.conf file with nano or your preferred editor.
```
sudo nano /etc/wireguard/wg0.conf
```
At the bottom of the file after the SaveConfig = true line, paste the following lines:
```
PostUp = ufw route allow in on wg0 out on eth0 
PostUp = iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE 
PostUp = ip6tables -t nat -I POSTROUTING -o eth0 -j MASQUERADE 
PreDown = ufw route delete allow in on wg0 out on eth0 
PreDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE 
PreDown = ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```
Replace eth0 with your network interface name.

The PostUp lines will run when the WireGuard Server starts the virtual VPN tunnel. In the example here, it will add three ufw and iptables rules:

ufw route allow in on wg0 out on eth0 — This rule will allow forwarding IPv4 and IPv6 traffic that comes in on the wg0 VPN interface to the eth0 network interface on the server. It works in conjunction with the net.ipv4.ip_forward and net.ipv6.conf.all.forwarding sysctl values that you configured in the previous section.
iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE — This rule configures masquerading and rewrites IPv4 traffic that comes in on the wg0 VPN interface to make it appear like it originates directly from the WireGuard Server’s public IPv4 address.
ip6tables -t nat -I POSTROUTING -o eth0 -j MASQUERADE — This rule configures masquerading and rewrites IPv6 traffic that comes in on the wg0 VPN interface to make it appear like it originates directly from the WireGuard Server’s public IPv6 address.
The PreDown rules run when the WireGuard Server stops the virtual VPN tunnel. These rules are the inverse of the PostUp rules, and function to undo the forwarding and masquerading rules for the VPN interface when the VPN is stopped.

In both cases, edit the configuration to include or exclude the IPv4 and IPv6 rules that are appropriate for your VPN. For example, if you are just using IPv4, then you can exclude the lines with the ip6tables commands.

Conversely, if you are only using IPv6, then edit the configuration to only include the ip6tables commands. The ufw lines should exist for any combination of IPv4 and IPv6 networks. Save and close the file when you are finished.

The last part of configuring the firewall on your WireGuard Server is to allow traffic to and from the WireGuard UDP port itself. If you did not change the port in the server’s /etc/wireguard/wg0.conf file, the port that you will open is 51820. If you chose a different port when editing the configuration be sure to substitute it in the following UFW command.

In case you forgot to open the SSH port when following the prerequisite tutorial, add it here too:
```
sudo ufw allow 51820/udp 
sudo ufw allow OpenSSH
```
[Note: If you are using a different firewall or have customized your UFW configuration, you may need to add additional firewall rules. For example, if you decide to tunnel all of your network traffic over the VPN connection, you will need to ensure that port 53 traffic is allowed for DNS requests, and ports 80 and 443 for HTTP and HTTPS traffic respectively. If there are other protocols that you are using over the VPN then you will need to add rules for them as well.]


After adding those rules, disable and re-enable UFW to restart it and load the changes from all of the files you’ve modified:
```
sudo ufw disable 
sudo ufw enable
```
Leave UFW configuration steps in case it is disabled on your server.


# Starting the Wireguard Server:

WireGuard can be configured to run as a systemd service using its built-in wg-quick script. While you could manually use the wg command to create the tunnel every time you want to use the VPN, doing so is a manual process that becomes repetitive and error-prone. Instead, you can use Systemctl to manage the tunnel with the help of the wg-quick script.

Using a systemd service means that you can configure WireGuard to start up at boot so that you can connect to your VPN at any time as long as the server is running. To do this, enable the wg-quick service for the wg0 tunnel that you’ve defined by adding it to systemctl:
```
sudo systemctl enable wg-quick@wg0.service
```
[Notice that the command specifies the name of the tunnel wg0 device name as a part of the service name. This name maps to the /etc/wireguard/wg0.conf configuration file. This approach to naming means that you can create as many separate VPN tunnels as you would like using your server.

For example, you could have a tunnel device and the name of prod, and its configuration file would be /etc/wireguard/prod.conf. Each tunnel configuration can contain different IPv4, IPv6, and client firewall settings. In this way you can support multiple different peer connections, each with their own unique IP addresses and routing rules.]

Now start the service:
```
sudo systemctl start wg-quick.service
```
Double-check that the WireGuard service is active with the following command. You should see active (running) in the output:
```
sudo systemctl status wg-quick.service
```
[
Output:
● wg-quick@wg0.service - WireGuard via wg-quick(8) for wg0
Loaded: loaded (/lib/systemd/system/wg-quick@.service; enabled; vendor preset: enabled)
Active: active (exited) since Wed 2021–08–25 15:24:14 UTC; 5s ago
Docs: man:wg-quick(8)
man:wg(8)
https://www.wireguard.com/
https://www.wireguard.com/quickstart/
https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8
https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8
Process: 3245 ExecStart=/usr/bin/wg-quick up wg0 (code=exited, status=0/SUCCESS)
Main PID: 3245 (code=exited, status=0/SUCCESS)
Aug 25 15:24:14 wg0 wg-quick[3245]: [#] wg setconf wg0 /dev/fd/63
Aug 25 15:24:14 wg0 wg-quick[3245]: [#] ip -4 address add 10.8.0.1/24 dev wg0
Aug 25 15:24:14 wg0 wg-quick[3245]: [#] ip -6 address add fd0d:86fa:c3bc::1/64 dev wg0
Aug 25 15:24:14 wg0 wg-quick[3245]: [#] ip link set mtu 1420 up dev wg0
Aug 25 15:24:14 wg0 wg-quick[3245]: [#] ufw route allow in on wg0 out on eth0
Aug 25 15:24:14 wg0 wg-quick[3279]: Rule added
Aug 25 15:24:14 wg0 wg-quick[3279]: Rule added (v6)
Aug 25 15:24:14 wg0 wg-quick[3245]: [#] iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE
Aug 25 15:24:14 wg0 wg-quick[3245]: [#] ip6tables -t nat -I POSTROUTING -o eth0 -j MASQUERADE
Aug 25 15:24:14 wg0 systemd[1]: Finished WireGuard via wg-quick(8) for wg0.
]
The output shows the ip commands that are used to create the virtual wg0 device and assign it the IPv4 and IPv6 addresses that you added to the configuration file. You can use these rules to troubleshoot the tunnel, or with the wg command itself if you would like to try manually configuring the VPN interface.

# Configuring a Wireguard Peer (Client):

Install wireguard on the client:
```
sudo apt update 
sudo apt install wireguard
```
Creating the WireGuard Peer’s Key Pair

Next, you’ll need to generate the key pair on the peer using the same steps as you used on the server.

Private Key:

From your local machine or remote server that will serve as peer, proceed and create the private key for the peer using the following commands:
```
wg genkey | sudo tee /etc/wireguard/private.key 
sudo chmod go= /etc/wireguard/private.key
```
Public Key:
```
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
```
Creating the WireGuard Peer’s Configuration File

Now that you have a key pair, you can create a configuration file for the peer that contains all the information that it needs to establish a connection to the WireGuard Server.

You will need a few pieces of information for the configuration file:

The base64 encoded private key that you generated on the peer.
The IPv4 and IPv6 address ranges that you defined on the WireGuard Server.
The base64 encoded public key from the WireGuard Server.
The public IP address and port number of the WireGuard Server. Usually, this will be the IPv4 address, but if your server has an IPv6 address and your client machine has an IPv6 connection to the internet you can use this instead of IPv4.
With all this information at hand, open a new /etc/wireguard/wg0.conf file on the WireGuard Peer machine using Nano or your preferred editor:
```
sudo nano /etc/wireguard/wg0.conf
```
Add the following lines to the file, substituting the various data into the highlighted sections as required:
```
[Interface] 
PrivateKey = base64_encoded_peer_private_key_goes_here (Peer’s private key) 
Address = 10.0.0.2/24 
Address = fd0d:86fa:c3bc::2/64 (Option for IPV6) 

[Peer] 
PublicKey = U9uE2kb/nrrzsEU58GD3pKFU3TLYDMCbetIsnV8eeFE= (Server's public key) 
AllowedIPs = 10.0.0.0/24 
Endpoint = 43.205.212.66:51820 (Server's public IP)
```
Notice how the first Address line uses an IPv4 address from the 10.8.0.0/24 subnet that you chose earlier. This IP address can be anything in the subnet as long as it is different from the server’s IP. Incrementing addresses by 1 each time you add a peer is generally the easiest way to allocate IPs.

The other notable part of the file is the last AllowedIPs line. These two IPv4 and IPv6 ranges instruct the peer to only send traffic over the VPN if the destination system has an IP address in either range. Using the AllowedIPs directive, you can restrict the VPN on the peer to only connect to other peers and services on the VPN, or you can configure the setting to tunnel all traffic over the VPN and use the WireGuard Server as a gateway.

If you are only using IPv4, then omit the trailing fd0d:86fa:c3bc::/64 range (including the , comma). Conversely, if you are only using IPv6, then only include the fd0d:86fa:c3bc::/64 prefix and leave out the 10.8.0.0/24 IPv4 range.

In both cases, if you would like to send all your peer’s traffic over the VPN and use the WireGuard Server as a gateway for all traffic, then you can use 0.0.0.0/0, which represents the entire IPv4 address space, and ::/0 for the entire IPv6 address space.

Connecting the WireGuard Peer to the Tunnel:

Now that your server and peer are both configured to support your choice of IPv4, IPv6, packet forwarding, and DNS resolution, it is time to connect the peer to the VPN tunnel.

Since you may only want the VPN to be on for certain use cases, we’ll use the wg-quick command to establish the connection manually. If you would like to automate starting the tunnel like you did on the server, follow those steps in Step Starting the WireGuard Server section instead of using the wq-quick command.

In case you are routing all traffic through the VPN and have set up DNS forwarding, you’ll need to install the resolvconf utility on the WireGuard Peer before you start the tunnel. Run the following command to set this up:
```
sudo apt install resolvconf
```
To start the tunnel, run the following on the WireGuard Peer:
```
sudo wg-quick up wg0
```
Output:
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.8.0.2/24 dev wg0
[#] ip -6 address add fd0d:86fa:c3bc::2/64 dev wg0
[#] ip link set mtu 1420 up dev wg0
[#] resolvconf -a tun.wg0 -m 0 -x
Notice the highlighted IPv4 and IPv6 addresses that you assigned to the peer.

If you set the AllowedIPs on the peer to 0.0.0.0/0 and ::/0 (or to use ranges other than the ones that you chose for the VPN), then your output will resemble the following:
```
sudo wg-quick up wg0
```
Output:
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.8.0.2/24 dev wg0
[#] ip -6 address add fd0d:86fa:c3bc::2/64 dev wg0
[#] ip link set mtu 1420 up dev wg0
[#] resolvconf -a tun.wg0 -m 0 -x
[#] wg set wg0 fwmark 51820
[#] ip -6 route add ::/0 dev wg0 table 51820
[#] ip -6 rule add not fwmark 51820 table 51820
[#] ip -6 rule add table main suppress_prefixlength 0
[#] ip6tables-restore -n
[#] ip -4 route add 0.0.0.0/0 dev wg0 table 51820
[#] ip -4 rule add not fwmark 51820 table 51820
[#] ip -4 rule add table main suppress_prefixlength 0
[#] sysctl -q net.ipv4.conf.all.src_valid_mark=1
[#] iptables-restore –n

In this example, notice the highlighted routes that the command added, which correspond to the AllowedIPs in the peer configuration.

You can check the status of the tunnel on the peer using the wg command:
```
sudo wg
```

Output:
interface: wg0
public key: PeURxj4Q75RaVhBKkRTpNsBPiPSGb5oQijgJsTa29hg=
private key: (hidden)
listening port: 49338
fwmark: 0xca6c
peer: U9uE2kb/nrrzsEU58GD3pKFU3TLYDMCbetIsnV8eeFE=
endpoint: 203.0.113.1:51820
allowed ips: 10.8.0.0/24, fd0d:86fa:c3bc::/64
latest handshake: 1 second ago
transfer: 6.50 KiB received, 15.41 KiB sent
Once you are ready to disconnect from the VPN on the peer, use the wg-quick command:
```
sudo wg-quick down wg0
```
Output
[#] ip link delete dev wg0
[#] resolvconf -d tun.wg0 -f
If you would like to remove a peer’s configuration from the WireGuard Server completely, you can run the following command, being sure to substitute the correct public key for the peer that you want to remove:
```
sudo wg set wg0 peer PeURxj4Q75RaVhBKkRTpNsBPiPSGb5oQijgJsTa29hg= allowed-ips 10.0.0.2
```
Typically you will only need to remove a peer configuration if the peer no longer exists, or if its encryption keys are compromised or changed. Otherwise, it is better to leave the configuration in place so that the peer can reconnect to the VPN without requiring you to add its key and allowed IPs each time.


This setup allows you to securely connect to your private network from anywhere in the world. WireGuard’s simplicity, speed, and security make it an excellent choice for VPN solutions.
