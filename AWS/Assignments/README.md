# Building a Cloud Network From Zero — AWS VPC & Networking

**Region:** eu-west-2 (London) &nbsp;|&nbsp; **Console:** AWS Management Console &nbsp;|&nbsp; **Type:** Networking / Infrastructure

---

No default VPC. No shortcuts. Every component - subnets, routes, gateways, firewall rules — deliberately designed and built by hand. This project is about understanding what actually happens beneath the surface of a cloud network before anything else gets built on top of it.

---

## What This Project Covers

Most cloud engineers interact with networking after the fact — adjusting rules, adding subnets, wondering why traffic isn't flowing. This project flips that. Starting from a blank VPC, every decision was made intentionally: where resources live, how they communicate, what they're allowed to do, and crucially - what they're **not** allowed to do.

The end result is a segmented, production-aligned network with a public-facing tier and a fully isolated private tier, connected only through controlled, one-directional gateways.

---

## Network Layout

```
                          Internet
                             │
                      ┌──────▼──────┐
                      │     IGW     │  ← two-way internet access
                      └──────┬──────┘
                             │
                ┌────────────▼─────────────────┐
                │           Project-VPC         │
                │                               │
                │   ┌───────────────────────┐   │
                │   │     Public Subnet     │   │
                │   │                       │   │
                │   │   Project-Public-EC2  │   │
                │   │   10.0.0.173          │   │
                │   │   Public IP assigned  │   │
                │   └──────────┬────────────┘   │
                │              │                │
                │       ┌──────▼──────┐         │
                │       │ NAT Gateway │         │  ← outbound only
                │       └──────┬──────┘         │
                │              │                │
                │   ┌──────────▼────────────┐   │
                │   │     Private Subnet    │   │
                │   │                       │   │
                │   │  Project-Private-EC2  │   │
                │   │  10.0.1.35            │   │
                │   │  No public IP         │   │
                │   └───────────────────────┘   │
                └───────────────────────────────┘
```

The public subnet connects outward through the **Internet Gateway** - full two-way traffic. The private subnet routes outbound requests through a **NAT Gateway**, meaning it can reach the internet (for things like package installs or updates), but nothing from the internet can initiate a connection back in. That asymmetry is the whole point.

---

## Infrastructure Breakdown

### Compute

| Instance | Subnet | Public IP | Private IP | Type |
|----------|--------|-----------|------------|------|
| Project-Public-EC2 | Public | `35.179.186.193` | `10.0.0.173` | t3.micro |
| Project-Private-EC2 | Private | — | `10.0.1.35` | t3.micro |

The private instance has no public IP by design. The only way in is through the public EC2, which acts as a jump host - a deliberate implementation of the **bastion host pattern** used in real production environments.

### Routing

Two separate route tables were created - one per subnet. The public route table sends `0.0.0.0/0` traffic to the IGW. The private route table sends `0.0.0.0/0` to the NAT Gateway. This explicit separation means the private subnet cannot accidentally gain internet exposure through a misconfigured route.

### Firewall Rules — `Project-Public-SG`

| Direction | Protocol | Port | Source | Why |
|-----------|----------|------|--------|-----|
| Inbound | TCP | 22 | My IP only | SSH — scoped to a single trusted source |
| Inbound | TCP | 80 | `0.0.0.0/0` | HTTP — intentionally public |
| Outbound | All | All | `0.0.0.0/0` | Unrestricted outbound |

SSH is not open to the world. It never should be. Port 22 is locked to a single IP - a small detail that makes a significant difference in real exposure.

---

## Why Each Component Exists

**Internet Gateway** - Without this, nothing in the VPC can reach the internet and nothing from the internet can reach in. It's the VPC's front door, but it only opens for resources in the public subnet with a public IP.

**NAT Gateway** - The private subnet needs to be able to pull updates, reach external APIs, or install packages. NAT allows that without creating any inbound attack surface. Traffic flows out, responses come back, but no unsolicited inbound connection can land.

**Separate Route Tables** - Using a single default route table for everything is how misconfigurations happen. Explicit per-subnet routing makes the network's intent readable and auditable.

**Least-Privilege Security Groups** - Security groups are the last line of defence at the instance level. Every rule has a reason. Nothing is open "just in case."

---

## Evidence

## 📸 Evidence

### VPC Details
> `Project-VPC` — `vpc-0900fb6efc2fee01d` | CIDR `10.0.0.0/16`

![image](./project-1-vpc.png)

---

### Subnets
> Public (`10.0.0.0/24`) and Private (`10.0.1.0/24`) subnets within `Project-VPC`

![Subnets](screenshots/subnets.png)

---

### NAT Gateway
> Deployed in the public subnet with an Elastic IP — handles outbound traffic for the private subnet

![NAT Gateway](screenshots/nat-gateway.png)

---

### Internet Gateway
> Attached to `Project-VPC` — enables two-way internet access for the public subnet

![Internet Gateway](screenshots/igw.png)

---

### EC2 Instances
> `Project-Public-EC2` (public IP assigned) vs `Project-Private-EC2` (no public IP)

![Public EC2](screenshots/ec2-public.png)
![Private EC2](screenshots/ec2-private.png)

---

### Security Group
> `Project-Public-SG` — least privilege inbound rules (SSH scoped to My IP, HTTP open)

![Security Group](screenshots/sg.png)

---

## Stack

`Amazon VPC` `EC2 (t3.micro)` `Internet Gateway` `NAT Gateway` `Route Tables` `Security Groups` `Public & Private Subnets`

---

> Built manually via the AWS Console — eu-west-2 (London). Infrastructure deprovisioned post-completion.