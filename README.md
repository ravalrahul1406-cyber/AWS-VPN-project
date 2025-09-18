AWS Site-to-Site VPN Lab with StrongSwan
This lab demonstrates how to simulate a Site-to-Site VPN in AWS using two VPCs and StrongSwan.

Lab Architecture
Cloud VPC (AWS side): 10.0.0.0/16

Public subnet: 10.0.1.0/24
Private subnet: 10.0.2.0/24
On-Prem VPC (simulated): 192.168.0.0/16

Public subnet: 192.168.1.0/24
Region: Same region for simplicity

EC2 Instances:

StrongSwan (On-Prem VPC) → Ubuntu 22.04 LTS, t2.micro
Cloud Test EC2 → Ubuntu in 10.0.2.0/24
Security Groups:

Allow: UDP 500, UDP 4500, ESP (protocol 50), ICMP, SSH
Step 1 — Create VPCs
Cloud-VPC: 10.0.0.0/16

Subnets: 10.0.1.0/24 (public), 10.0.2.0/24 (private)
Attach Internet Gateway, add route 0.0.0.0/0 → IGW (public subnet)
OnPrem-VPC: 192.168.0.0/16

Subnet: 192.168.1.0/24
Attach Internet Gateway, add route 0.0.0.0/0 → IGW
Step 2 — Launch EC2 Instances
A. StrongSwan EC2 (OnPrem-VPC)

Ubuntu 22.04, t2.micro
Security group: SSH (22), UDP 500/4500, ESP (50), ICMP
B. Test EC2 (Cloud-VPC)

Ubuntu in 10.0.2.0/24
Security group: SSH, ICMP
Step 3 — Allocate Elastic IP
Associate Elastic IP with StrongSwan EC2.
Use this as Customer Gateway IP.
Step 4 — Create Customer Gateway (CGW)
Name: CGW-StrongSwan
BGP ASN: 65000
IP Address: Elastic IP of StrongSwan
Step 5 — Create Virtual Private Gateway (VGW)
Attach VGW to Cloud-VPC
Step 6 — Create Site-to-Site VPN Connection
Gateway type: Virtual Private Gateway
Customer Gateway: CGW-StrongSwan
Routing: Static → On-Prem CIDR: 192.168.0.0/16
Download VPN config (StrongSwan)
Step 7 — Configure StrongSwan
sudo apt update
sudo apt install -y strongswan strongswan-pki iptables-persistent
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
Disable Source/Dest Check on EC2.

/etc/ipsec.conf example:

conf
Copy code
config setup
  charondebug="ike 2, knl 2, cfg 2, net 2"

conn %default
  keyexchange=ikev2
  authby=psk
  rekey=no
  dpdaction=clear
  dpddelay=300s

conn aws-tunnel1
  left=%any
  leftid=<Elastic IP>
  leftsubnet=192.168.0.0/16
  right=<AWS VPN Outside IP>
  rightid=<AWS VPN Outside IP>
  rightsubnet=10.0.0.0/16
  auto=start
/etc/ipsec.secrets

conf
Copy code
: PSK "PSK_FROM_AWS_TUNNEL_1"
Restart StrongSwan:

bash
Copy code
sudo systemctl restart strongswan
sudo ipsec statusall
Step 8 — Firewall Rules
bash
Copy code
sudo iptables -A INPUT -p udp --dport 500 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 4500 -j ACCEPT
sudo iptables -A INPUT -p 50 -j ACCEPT
sudo iptables -A INPUT -p icmp -j ACCEPT
sudo iptables -A FORWARD -s 192.168.0.0/16 -d 10.0.0.0/16 -j ACCEPT
sudo iptables -A FORWARD -s 10.0.0.0/16 -d 192.168.0.0/16 -j ACCEPT
sudo netfilter-persistent save
Step 9 — Update Route Tables
Cloud VPC → Route Table → Add route: 192.168.0.0/16 → VGW

On-Prem VPC → Route Table → Add route: 10.0.0.0/16 → StrongSwan EC2

Step 10 — Test Connectivity
From Cloud EC2: ping 192.168.1.x

From On-Prem EC2: ping 10.0.2.x

Check StrongSwan: sudo ipsec statusall
