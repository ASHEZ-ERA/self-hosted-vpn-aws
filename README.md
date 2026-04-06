# Self-Hosted VPN Server on AWS EC2 (OpenVPN)

> Provisioned and configured a production-grade VPN server on AWS EC2 from scratch — no third-party VPN service used.

---

## Project Overview

Built a fully functional VPN gateway on an AWS EC2 instance (Ubuntu) using **OpenVPN** and **Easy-RSA**. The VPN tunnels all client traffic through the EC2 server, masking the client's public IP with the server's AWS IP. A private Windows EC2 instance (no public IP) was then accessed and used to host a website — reachable exclusively through the VPN.

**Outcome:** Client machine's public IP changed from local ISP to `3.108.41.168` (AWS Mumbai) — verified via [whatismyipaddress.com](https://whatismyipaddress.com).

---

## Architecture

```
Your Laptop  ──[OpenVPN tunnel]──►  EC2 Ubuntu Server (Public IP: 3.108.41.168)
                                         │
                                         │  NAT / IP Masquerading
                                         │  (iptables POSTROUTING)
                                         ▼
                                    Internet / Private VPC Resources
                                         │
                                         ▼
                              Windows EC2 (Private IP only: 172.31.39.246)
                              [IIS Web Server — accessible via VPN only]
```

---

## Skills & Technologies Used

| Category | Tools |
|---|---|
| Cloud | AWS EC2, AWS Security Groups, VPC |
| Networking | OpenVPN, PKI/TLS, Diffie-Hellman, NAT, IP Masquerading |
| Linux | Ubuntu 22.04, systemctl, iptables, UFW, sysctl, ip routing |
| Security | Easy-RSA, Certificate Authority (CA), CSR signing, TLS auth key |
| Protocols | UDP, IPv4 forwarding, DHCP-push, DNS push |

---

## What I Built — Step by Step

### Phase 1: Certificate Infrastructure (PKI)
- Installed `openvpn` and `easy-rsa` on the EC2 instance
- Initialized a PKI directory and created a **Certificate Authority (CA)**
- Generated Certificate Signing Requests (CSRs) for the server and client
- Signed both CSRs with the CA — creating trusted server and client certificates
- Generated **Diffie-Hellman parameters** (`dh.pem`) for the initial key exchange
- Generated a **TLS Auth key** (`ta.key`) for an additional layer of HMAC authentication

### Phase 2: Server Configuration
- Wrote a `server.conf` with:
  - UDP on port `1111`
  - Virtual network `172.31.0.0/255.255.255.0`
  - Full traffic redirect (`push "redirect-gateway def1"`)
  - DNS push to Google (`8.8.8.8`)
  - AES-256-CBC cipher, keepalive, and duplicate-cn support
- Enabled **IP forwarding** in the Linux kernel (`net.ipv4.ip_forward=1`) and made it persistent via `/etc/sysctl.conf`

### Phase 3: Firewall & NAT Rules
- Configured `iptables` POSTROUTING with **MASQUERADE** to NAT VPN client traffic through the server's public IP
- Persisted the NAT rule in `/etc/ufw/before.rules`
- Configured UFW:
  - Allowed port `1111/udp` for VPN traffic
  - Allowed port `22/tcp` to prevent SSH lockout
  - Set `DEFAULT_FORWARD_POLICY=ACCEPT`

### Phase 4: Client & Verification
- Transferred certificates and keys securely via **WinSCP**
- Created a `.ovpn` client profile referencing the CA cert, client cert, client key, and TLS auth key
- Imported profile into **OpenVPN Connect** on Windows
- Verified IP change: client's public IP became the EC2 server's IP (`3.108.41.168`)
- Connected via **RDP** to a private Windows EC2 instance (no public IP) — accessible only through the VPN tunnel
- Accessed an **IIS web server** hosted on the private Windows instance over the VPN

---

## Key Concepts Demonstrated

- **PKI & Certificate Authority**: Understood how trust chains work — CA signs server/client certs, OpenVPN validates both ends
- **Diffie-Hellman Key Exchange**: Enables secure session key negotiation without transmitting the key itself
- **NAT / IP Masquerading**: Linux kernel replaces private VPN IPs with the server's public IP so internet routing works
- **Split vs Full Tunnel**: Configured full-tunnel mode — all client traffic routes through the VPN
- **Defense in Depth**: Two firewall layers — AWS Security Groups (outer) + UFW (inner)

---

## Files in This Repository

```
├── server.conf          # OpenVPN server configuration
├── README.md            # This file
└── docs/
    └── VPN-Using-openVPNn.pdf   # Full project walkthrough with screenshots
```

> **Note:** Private keys, certificates, and `.ovpn` profiles are intentionally excluded from this repository. Never commit cryptographic key material to version control.

---

## How to Reproduce This Project

**Prerequisites:** AWS account, an EC2 Ubuntu instance (t2.micro works), port 1111/UDP open in Security Group, and OpenVPN Connect on your client machine.

```bash
# 1. Install tools
sudo apt-get install -y openvpn easy-rsa

# 2. Set up PKI
make-cadir easy-rsa && cd easy-rsa
./easyrsa init-pki
./easyrsa build-ca
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-req client nopass
./easyrsa sign-req client client
./easyrsa gen-dh
openvpn --genkey secret ta.key

# 3. Copy files to OpenVPN directory
sudo cp pki/ca.crt pki/issued/server.crt pki/private/server.key pki/dh.pem ~/easy-rsa/ta.key /etc/openvpn/

# 4. Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf

# 5. Add NAT rule
sudo iptables -t nat -A POSTROUTING -s 172.31.0.0/24 -o ens5 -j MASQUERADE

# 6. Configure UFW
sudo ufw allow 1111/udp
sudo ufw allow 22/tcp
sudo ufw enable

# 7. Start and enable OpenVPN
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
```

See `server.conf` for the full server configuration and `docs/` for a complete visual walkthrough.

---

## Result

| Before VPN | After VPN |
|---|---|
| Local ISP IP | `3.108.41.168` (AWS Mumbai) |
| No access to private VPC resources | Full access to private Windows EC2 |
| Public traffic | Encrypted tunnel through EC2 |

---

*Project completed— March 2026*
*Author: Dawood Ashraf*
