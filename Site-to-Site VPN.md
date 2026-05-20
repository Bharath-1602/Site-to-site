# Perfect AWS Site-to-Site VPN — Complete Lab From Scratch

---

## Architecture We Are Building

```
ON-PREM VPC (10.10.0.0/16)          AWS CLOUD VPC (10.20.0.0/16)
                    
┌──────────────────────┐              ┌──────────────────────┐
│  Public Subnet       │              │  Private Subnet      │
│  10.10.1.0/24        │              │  10.20.1.0/24        │
│                      │              │                      │
│  ┌────────────────┐  │              │  ┌────────────────┐  │
│  │ VPN Appliance  │  │◄────────────►│  │  AWS Private   │  │
│  │ StrongSwan     │  │  IPSec ESP   │  │  EC2           │  │
│  │ 10.10.1.x      │  │  AES-128     │  │  10.20.1.x     │  │
│  └────────────────┘  │              │  └────────────────┘  │
│                      │              │                      │
│  Private Subnet      │              │  Virtual Private     │
│  10.10.2.0/24        │              │  Gateway (VGW)       │
│  ┌────────────────┐  │              └──────────────────────┘
│  │ OnPrem Private │  │
│  │ EC2            │  │
│  │ 10.10.2.x      │  │
│  └────────────────┘  │
└──────────────────────┘
```

---

## PART 1 — CREATE ON-PREM VPC

### Step 1 — Create VPC

```
AWS Console
→ VPC
→ Your VPCs
→ Create VPC
```

| Field | Value |
|-------|-------|
| Name tag | OnPrem-VPC |
| IPv4 CIDR | 10.10.0.0/16 |
| Tenancy | Default |

```
→ Create VPC
```

---

### Step 2 — Create Internet Gateway for OnPrem VPC

```
VPC
→ Internet Gateways
→ Create Internet Gateway
```

| Field | Value |
|-------|-------|
| Name | OnPrem-IGW |

```
→ Create
→ Actions
→ Attach to VPC
→ Select OnPrem-VPC
→ Attach
```

---

### Step 3 — Create Public Subnet in OnPrem VPC

```
VPC
→ Subnets
→ Create Subnet
```

| Field | Value |
|-------|-------|
| VPC | OnPrem-VPC |
| Subnet name | OnPrem-Public-Subnet |
| Availability Zone | Pick any one (e.g. ap-south-1a) |
| IPv4 CIDR | 10.10.1.0/24 |

```
→ Create Subnet
```

---

### Step 4 — Create Private Subnet in OnPrem VPC

```
VPC
→ Subnets
→ Create Subnet
```

| Field | Value |
|-------|-------|
| VPC | OnPrem-VPC |
| Subnet name | OnPrem-Private-Subnet |
| Availability Zone | Same as above |
| IPv4 CIDR | 10.10.2.0/24 |

```
→ Create Subnet
```

---

### Step 5 — Create Public Route Table for OnPrem

```
VPC
→ Route Tables
→ Create Route Table
```

| Field | Value |
|-------|-------|
| Name | OnPrem-Public-RT |
| VPC | OnPrem-VPC |

```
→ Create
```

**Add Internet Route:**

```
→ Select OnPrem-Public-RT
→ Routes tab
→ Edit Routes
→ Add Route

Destination: 0.0.0.0/0
Target: Internet Gateway → OnPrem-IGW

→ Save
```

**Associate Public Subnet:**

```
→ Subnet Associations tab
→ Edit Subnet Associations
→ Check OnPrem-Public-Subnet
→ Save
```

---

### Step 6 — Create Private Route Table for OnPrem

```
VPC
→ Route Tables
→ Create Route Table
```

| Field | Value |
|-------|-------|
| Name | OnPrem-Private-RT |
| VPC | OnPrem-VPC |

```
→ Create
```

**Associate Private Subnet:**

```
→ Subnet Associations tab
→ Edit Subnet Associations
→ Check OnPrem-Private-Subnet
→ Save
```

**NOTE — Do NOT add any routes here yet. We add VPN route later.**

---

## PART 2 — CREATE AWS CLOUD VPC

### Step 7 — Create AWS Cloud VPC

```
VPC
→ Your VPCs
→ Create VPC
```

| Field | Value |
|-------|-------|
| Name tag | AWS-Cloud-VPC |
| IPv4 CIDR | 10.20.0.0/16 |
| Tenancy | Default |

```
→ Create VPC
```

---

### Step 8 — Create Private Subnet in AWS Cloud VPC

```
VPC
→ Subnets
→ Create Subnet
```

| Field | Value |
|-------|-------|
| VPC | AWS-Cloud-VPC |
| Subnet name | AWS-Private-Subnet |
| Availability Zone | Pick any one |
| IPv4 CIDR | 10.20.1.0/24 |

```
→ Create Subnet
```

---

### Step 9 — Create Route Table for AWS Cloud VPC

```
VPC
→ Route Tables
→ Create Route Table
```

| Field | Value |
|-------|-------|
| Name | AWS-Cloud-Private-RT |
| VPC | AWS-Cloud-VPC |

```
→ Create
```

**Associate AWS Private Subnet:**

```
→ Subnet Associations tab
→ Edit Subnet Associations
→ Check AWS-Private-Subnet
→ Save
```

**NOTE — Do NOT add routes yet. VGW route comes later.**

---

## PART 3 — LAUNCH EC2 INSTANCES

### Step 10 — Launch VPN Appliance EC2

```
EC2
→ Instances
→ Launch Instance
```

| Field | Value |
|-------|-------|
| Name | VPN-Appliance |
| AMI | Ubuntu Server 22.04 LTS |
| Instance Type | t2.micro |
| Key Pair | Create new or use existing |
| VPC | OnPrem-VPC |
| Subnet | OnPrem-Public-Subnet |
| Auto-assign Public IP | DISABLE (we use Elastic IP) |

**Create Security Group:**

```
Name: VPN-Appliance-SG

Inbound Rules:
─────────────────────────────────────────
Type          Protocol  Port   Source
─────────────────────────────────────────
SSH           TCP       22     0.0.0.0/0
All ICMP-v4   ICMP      All    0.0.0.0/0
Custom UDP    UDP       4500   0.0.0.0/0
Custom UDP    UDP       500    0.0.0.0/0
All Traffic   All       All    0.0.0.0/0
─────────────────────────────────────────

Outbound Rules:
─────────────────────────────────────────
Type          Protocol  Port   Destination
─────────────────────────────────────────
All Traffic   All       All    0.0.0.0/0
─────────────────────────────────────────
```

```
→ Launch Instance
```

---

### Step 11 — Assign Elastic IP to VPN Appliance

```
EC2
→ Elastic IPs
→ Allocate Elastic IP Address
→ Allocate
```

```
→ Select the new Elastic IP
→ Actions
→ Associate Elastic IP Address
→ Instance → Select VPN-Appliance
→ Associate
```

**WRITE DOWN THIS ELASTIC IP — You need it later**

```
My Elastic IP: ___________________
```

---

### Step 12 — Disable Source/Destination Check on VPN Appliance

```
EC2
→ Instances
→ Select VPN-Appliance
→ Actions
→ Networking
→ Change Source/Destination Check
→ UNCHECK "Enable"
→ Save
```

**This is critical. Without this VPN appliance drops forwarded packets.**

---

### Step 13 — Launch OnPrem Private EC2

```
EC2
→ Launch Instance
```

| Field | Value |
|-------|-------|
| Name | OnPrem-Private-EC2 |
| AMI | Ubuntu Server 22.04 LTS |
| Instance Type | t2.micro |
| Key Pair | Same key pair |
| VPC | OnPrem-VPC |
| Subnet | OnPrem-Private-Subnet |
| Auto-assign Public IP | Disable |

**Create Security Group:**

```
Name: OnPrem-Private-SG

Inbound Rules:
─────────────────────────────────────────
Type              Protocol  Port  Source
─────────────────────────────────────────
All ICMP - IPv4   ICMP      All   0.0.0.0/0
SSH               TCP       22    0.0.0.0/0
─────────────────────────────────────────

Outbound Rules:
─────────────────────────────────────────
All Traffic   All   All   0.0.0.0/0
─────────────────────────────────────────
```

```
→ Launch Instance
```

---

### Step 14 — Launch AWS Private EC2

```
EC2
→ Launch Instance
```

| Field | Value |
|-------|-------|
| Name | AWS-Private-EC2 |
| AMI | Ubuntu Server 22.04 LTS |
| Instance Type | t2.micro |
| Key Pair | Same key pair |
| VPC | AWS-Cloud-VPC |
| Subnet | AWS-Private-Subnet |
| Auto-assign Public IP | Disable |

**Create Security Group:**

```
Name: AWS-Private-SG

Inbound Rules:
─────────────────────────────────────────
Type              Protocol  Port  Source
─────────────────────────────────────────
All ICMP - IPv4   ICMP      All   0.0.0.0/0
All Traffic       All       All   10.10.0.0/16
─────────────────────────────────────────

Outbound Rules:
─────────────────────────────────────────
All Traffic   All   All   0.0.0.0/0
─────────────────────────────────────────
```

```
→ Launch Instance
```

**WRITE DOWN THIS PRIVATE IP:**

```
AWS Private EC2 IP: ___________________
```

---

## PART 4 — CREATE VPN COMPONENTS

### Step 15 — Create Virtual Private Gateway

```
VPC
→ Virtual Private Gateways
→ Create Virtual Private Gateway
```

| Field | Value |
|-------|-------|
| Name | AWS-VGW |
| ASN | Amazon default ASN |

```
→ Create Virtual Private Gateway
```

**Attach to AWS Cloud VPC:**

```
→ Select AWS-VGW
→ Actions
→ Attach to VPC
→ Select AWS-Cloud-VPC
→ Attach
```

Wait for state to show **Attached**

---

### Step 16 — Create Customer Gateway

```
VPC
→ Customer Gateways
→ Create Customer Gateway
```

| Field | Value |
|-------|-------|
| Name | OnPrem-CGW |
| Routing | Static |
| IP Address | YOUR ELASTIC IP (from Step 11) |

```
→ Create Customer Gateway
```

---

### Step 17 — Create Site-to-Site VPN Connection

```
VPC
→ Site-to-Site VPN Connections
→ Create VPN Connection
```

| Field | Value |
|-------|-------|
| Name | OnPrem-to-AWS-VPN |
| Target Gateway Type | Virtual Private Gateway |
| Virtual Private Gateway | AWS-VGW |
| Customer Gateway | Existing |
| Customer Gateway ID | OnPrem-CGW |
| Routing Options | Static |

**Add Static Route:**

```
Static IP Prefixes: 10.10.0.0/16
```

```
→ Create VPN Connection
```

**Wait 2-3 minutes for VPN to be created.**

---

### Step 18 — Download VPN Configuration

```
→ Select your VPN Connection
→ Download Configuration
```

| Field | Value |
|-------|-------|
| Vendor | Generic |
| Platform | Generic |
| Software | Vendor Agnostic |

```
→ Download
```

**Open the downloaded file. Write down these values:**

```
Tunnel 1 Outside IP (AWS side):     ___________________
Tunnel 1 Pre-Shared Key:            ___________________
```

---

## PART 5 — UPDATE ROUTE TABLES

### Step 19 — Enable Route Propagation on AWS Cloud Route Table

```
VPC
→ Route Tables
→ Select AWS-Cloud-Private-RT
→ Route Propagation tab
→ Edit Route Propagation
→ ENABLE checkbox next to AWS-VGW
→ Save
```

---

### Step 20 — Add Route to AWS Cloud Route Table

```
VPC
→ Route Tables
→ Select AWS-Cloud-Private-RT
→ Routes tab
→ Edit Routes
→ Add Route

Destination: 10.10.0.0/16
Target: Virtual Private Gateway → AWS-VGW

→ Save
```

**Your AWS Cloud Route Table should now look like:**

| Destination | Target |
|------------|--------|
| 10.20.0.0/16 | local |
| 10.10.0.0/16 | vgw-xxxxxxxx |

---

### Step 21 — Add Route to OnPrem Private Route Table

```
VPC
→ Route Tables
→ Select OnPrem-Private-RT
→ Routes tab
→ Edit Routes
→ Add Route

Destination: 10.20.0.0/16
Target: Instance → Select VPN-Appliance EC2

→ Save
```

---

## PART 6 — CONFIGURE STRONGSWAN ON VPN APPLIANCE

### Step 22 — SSH Into VPN Appliance

```bash
ssh -i your-key.pem ubuntu@YOUR-ELASTIC-IP
```

---

### Step 23 — Install StrongSwan

```bash
sudo apt update
sudo apt install strongswan strongswan-starter -y
```

---
### Check IPsec version

```bash
ipsec version
```
---
### Step 24 — Enable IP Forwarding

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-ipforward.conf
```

### Check if it works
 
```bash
sysctl net.ipv4.ip_forward
```

**Must show: 1**

---

### Step 25 — Configure IPSec

```bash
sudo nano /etc/ipsec.conf
```

**Delete everything and paste exactly this:**

```
config setup
    charondebug="ike 1, knl 1, cfg 1"
    uniqueids=no

conn aws-vpn
    auto=start
    type=tunnel
    keyexchange=ikev1
    authby=psk

    left=%defaultroute
    leftid=YOUR_ELASTIC_IP
    leftsubnet=10.10.0.0/16

    right=AWS_TUNNEL1_OUTSIDE_IP
    rightsubnet=10.20.0.0/16

    ike=aes128-sha1-modp1024!
    esp=aes128-sha1-modp1024!

    ikelifetime=8h
    lifetime=1h
    margintime=5m

    dpdaction=restart
    dpddelay=10s
    dpdtimeout=30s
```

**Replace:**
- `YOUR_ELASTIC_IP` → Your actual Elastic IP
- `AWS_TUNNEL1_OUTSIDE_IP` → From downloaded config file

---

### Step 26 — Configure Pre-Shared Key

```bash
sudo nano /etc/ipsec.secrets
```

**Delete everything and paste exactly this:**

```
YOUR_ELASTIC_IP AWS_TUNNEL1_OUTSIDE_IP : PSK "PreSharedKeyFromConfigFile"
```

**Replace all three values from your downloaded config file.**

---

### Step 27 — Start StrongSwan

```bash
sudo systemctl restart strongswan-starter
sudo ipsec restart
sudo ipsec statusall
```

Wait 10 seconds then check:

```bash
sudo ipsec statusall
```

**You must see:**

```
aws-vpn[1]: ESTABLISHED
aws-vpn{1}: INSTALLED, TUNNEL
aws-vpn{1}: 10.10.0.0/16 === 10.20.0.0/16
```

---

### Step 28 — Add Correct Route on VPN Appliance

**This is the most important step — do NOT skip this.**

```bash
sudo ip route add 10.20.0.0/16 via 10.10.1.1 dev enX0

sudo ip route add 10.10.2.0/24 via 10.10.1.1 dev enX0
```

Verify:

```bash
ip route
```

Must show:

```
10.20.0.0/16 via 10.10.1.1 dev enX0
```

**NOT `scope link` — must have `via 10.10.1.1`**

---

## PART 7 — VERIFY EVERYTHING

### Step 29 — Full Verification Checklist

Run all these on VPN Appliance:

```bash
# Check 1 - IP forwarding enabled

```bash
sysctl net.ipv4.ip_forward

# Must show: 1

# Check 2 - Tunnel established
sudo ipsec statusall
# Must show: ESTABLISHED and INSTALLED

# Check 3 - Route is correct
ip route | grep 10.20
# Must show: 10.20.0.0/16 via 10.10.1.1 dev enX0

# Check 4 - SSH into OnPrem-Private-EC2
ssh -i yourkey.pem ubuntu@10.10.2.X (Private ip of onPrem-private-ec2)

# Check 5 - Ping AWS Private EC2
ping 10.20.1.x -c 5
# Must show: 0% packet loss
```

---

---

## COMPLETE VISUAL CHECKLIST

```
INFRASTRUCTURE:
□ OnPrem-VPC created (10.10.0.0/16)
□ OnPrem-IGW created and attached
□ OnPrem-Public-Subnet (10.10.1.0/24)
□ OnPrem-Private-Subnet (10.10.2.0/24)
□ OnPrem-Public-RT → IGW route → public subnet associated
□ OnPrem-Private-RT → private subnet associated
□ AWS-Cloud-VPC created (10.20.0.0/16)
□ AWS-Private-Subnet (10.20.1.0/24)
□ AWS-Cloud-Private-RT → AWS private subnet associated

EC2:
□ VPN-Appliance launched in OnPrem-Public-Subnet
□ Elastic IP attached to VPN-Appliance
□ Source/Destination Check DISABLED on VPN-Appliance
□ OnPrem-Private-EC2 launched in OnPrem-Private-Subnet
□ AWS-Private-EC2 launched in AWS-Private-Subnet

VPN COMPONENTS:
□ AWS-VGW created and attached to AWS-Cloud-VPC
□ OnPrem-CGW created with Elastic IP
□ VPN Connection created with static route 10.10.0.0/16
□ VPN config downloaded (Tunnel IP + PSK noted)

ROUTE TABLES:
□ AWS-Cloud-Private-RT has route: 10.10.0.0/16 → VGW
□ AWS-Cloud-Private-RT route propagation ENABLED
□ OnPrem-Private-RT has route: 10.20.0.0/16 → VPN-Appliance

STRONGSWAN:
□ StrongSwan installed
□ IP forwarding = 1
□ rp_filter = 0
□ ipsec.conf configured with correct IPs
□ ipsec.secrets configured with correct PSK
□ StrongSwan restarted
□ Tunnel shows ESTABLISHED
□ Tunnel shows INSTALLED

CRITICAL ROUTE:
□ ip route add 10.20.0.0/16 via 10.10.1.1 dev enX0
□ Route shows "via 10.10.1.1" NOT "scope link"

RESULT:
□ ping 10.20.1.x → 0% packet loss ✅
```

---

## Common Mistakes — Avoid These

| Mistake | Result | Fix |
|---------|--------|-----|
| `ip route add 10.20.0.0/16 dev enX0` without `via` | scope link, ARP fails, 0 packets | Always add `via 10.10.1.1` |
| Source/Dest check not disabled | Forwarded packets dropped silently | Disable it on VPN Appliance |
| Route propagation not enabled | AWS has no return path | Enable on AWS-Cloud-Private-RT |
| Wrong PSK in ipsec.secrets | IKE Phase 1 fails | Copy exactly from config file |
| Static route missing in VPN Connection | AWS doesnt know OnPrem subnet | Add 10.10.0.0/16 in VPN static routes |
| Security group blocks ICMP | Ping fails even if tunnel works | Allow All ICMP from both CIDRs |
| ufw active on AWS EC2 | Packets arrive but get dropped | sudo ufw disable |

---

**Follow this exactly tomorrow morning and it will work first time.**
