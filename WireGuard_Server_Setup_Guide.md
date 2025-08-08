# 🔐 WireGuard VPN Server Setup on Akamai Cloud Manager (Linode)

This guide outlines the full setup of a high-performance, secure WireGuard VPN server using Akamai Cloud Manager (formerly Linode). It walks you through installation, configuration, firewall setup, and adding clients — both Linux and Windows.

---

## ☁️ Step 1: Provision a Linode Server

1. Create a VM using Ubuntu (latest LTS is preferred).
2. SSH into your server:
   ```bash
   ssh root@<your-server-ip>
   ```

---

## ⚙️ Step 2: Install WireGuard

```bash
sudo apt update
sudo apt install wireguard wireguard-tools
```

---

## 🔑 Step 3: Generate Keys

```bash
cd /etc/wireguard/
wg genkey > privatekey
chmod 600 privatekey
wg pubkey < privatekey > publickey
```

Save your keys for future steps.

---

## ✏️ Step 4: Create the Server Config

Create the config file:
```bash
nano wg0.conf
```

Example contents:
```ini
[Interface]
Address = 10.10.10.1/24
SaveConfig = true
ListenPort = 51820
PrivateKey = <YOUR_SERVER_PRIVATE_KEY>
MTU = 1420
```

---

## 🚀 Step 5: Enable and Start WireGuard

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

---

## 🌐 Step 6: Enable IP Forwarding

```bash
nano /etc/sysctl.conf
```

Uncomment or add:
```
net.ipv4.ip_forward=1
```

Apply:
```bash
sudo sysctl -p
```

---

## 🔥 Step 7: Configure UFW Firewall

```bash
sudo ufw allow 51820/udp
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw default allow FORWARD
```

Edit NAT rules:
```bash
sudo nano /etc/ufw/before.rules
```

Add at the top:
```
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.10.10.0/24 -o eth0 -j MASQUERADE
COMMIT
```

Allow routing on `wg0`:
```bash
sudo ufw route allow in on wg0
sudo ufw reload
```

---

## 🔁 Step 8: Create WireGuard Client Config

```bash
mkdir /etc/wireguard/clients
cd /etc/wireguard/clients
wg genkey > client1key
chmod 600 client1key
wg pubkey < client1key > client1pub
```

Create client1.conf:
```ini
[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.10.10.2/32
DNS = 8.8.8.8
MTU = 1375

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <SERVER_PUBLIC_IP>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

---

## ➕ Step 9: Add Client to Server

```bash
wg set wg0 peer $(cat client1pub) allowed-ips 10.10.10.2/32
systemctl restart wg-quick@wg0
```

---

## 📱 Step 10: Use on Mobile (QR Code)

```bash
sudo apt install qrencode
cd /etc/wireguard/clients
qrencode -t PNG -o client1.png < client1.conf
```

Transfer the image to your phone and scan it using the WireGuard app.

---

## ✅ Verifying

Check connection:
```bash
sudo systemctl status wg-quick@wg0
sudo wg
```

Confirm port open:
```bash
sudo netstat -uln | grep 51820
```

List NAT rules:
```bash
sudo iptables -L -v -n -t nat
```

Ping across VPN to test.

---

## 🧪 Troubleshooting

- Check `journalctl -u wg-quick@wg0` for logs
- Confirm keys are correct
- Check firewall/NAT issues
- Use `sudo wg` to inspect live connections

---

## 🧑‍💻 Windows Client Use

1. Install WireGuard from [wireguard.com](https://www.wireguard.com/install/)
2. Import the `.conf` file or scan the QR code
3. Activate the tunnel
4. Confirm IP at [whatismyip.com](https://www.whatismyip.com)

---

## 📝 Notes

- Use `client2key`, `client3key` for additional clients
- Update `wg0.conf` or dynamically use `wg set`
- Adjust MTU if needed (1420–1375 is common)
