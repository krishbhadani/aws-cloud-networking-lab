# AWS Cloud Networking Lab

A hands-on lab building a **multi-tier, highly-available network architecture on AWS** — from scratch, using core VPC components. This is the standard reference pattern used by most companies for production workloads.

Built by **Krish Bhadani** as part of an ongoing learning project alongside a Bachelor of Information Technology (Networking & Cybersecurity) at Adelaide University.

---

## What This Project Demonstrates

- Designing an enterprise VPC layout with public/private subnet separation across two Availability Zones
- Routing traffic through the correct paths: Internet Gateway for public egress, NAT Gateway for outbound-only from private subnets
- Layered security using Security Groups with **security-group-as-source** references (identity-based segmentation)
- Deploying an Application Load Balancer for high availability across AZs
- Verifying architecture with **actual isolation and failover tests**, not just theory

---

## Architecture

```
                            Internet
                                │
                        ┌───────┴───────┐
                        │   Internet    │
                        │    Gateway    │
                        └───────┬───────┘
                                │
                     ┌──────────┴──────────┐
                     │  Application Load   │
                     │   Balancer (ALB)    │
                     └──────────┬──────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        │        VPC 10.0.0.0/16                        │
        │                                               │
        │  ┌──────────────────┐   ┌──────────────────┐  │
        │  │  Public Subnet   │   │  Public Subnet   │  │
        │  │   10.0.1.0/24    │   │   10.0.2.0/24    │  │
        │  │       AZ-A       │   │       AZ-B       │  │
        │  │                  │   │                  │  │
        │  │  ┌─────────────┐ │   │  ┌─────────────┐ │  │
        │  │  │ Web Server  │ │   │  │ Web Server  │ │  │
        │  │  │    EC2      │ │   │  │    EC2      │ │  │
        │  │  └─────────────┘ │   │  └─────────────┘ │  │
        │  │                  │   │                  │  │
        │  │  ┌─────────────┐ │   │                  │  │
        │  │  │ NAT Gateway │ │   │                  │  │
        │  │  └──────┬──────┘ │   │                  │  │
        │  └─────────┼────────┘   └──────────────────┘  │
        │            │                                   │
        │            ▼ (outbound only)                   │
        │  ┌──────────────────┐   ┌──────────────────┐  │
        │  │ Private Subnet   │   │ Private Subnet   │  │
        │  │   10.0.11.0/24   │   │   10.0.12.0/24   │  │
        │  │       AZ-A       │   │       AZ-B       │  │
        │  │                  │   │                  │  │
        │  │  ┌─────────────┐ │   │                  │  │
        │  │  │ App Server  │ │   │                  │  │
        │  │  │    EC2      │ │   │                  │  │
        │  │  └─────────────┘ │   │                  │  │
        │  └──────────────────┘   └──────────────────┘  │
        │                                               │
        └───────────────────────────────────────────────┘
```

---

## Components

### Networking

| Component | Purpose | Configuration |
|---|---|---|
| VPC | Isolated virtual network | `10.0.0.0/16` (65,536 addresses) |
| Public Subnet A | Web-tier + NAT Gateway | `10.0.1.0/24` in `ap-southeast-2a` |
| Public Subnet B | Web-tier (high availability) | `10.0.2.0/24` in `ap-southeast-2b` |
| Private Subnet A | App-tier | `10.0.11.0/24` in `ap-southeast-2a` |
| Private Subnet B | App-tier | `10.0.12.0/24` in `ap-southeast-2b` |
| Internet Gateway | Public egress + ingress | Attached to VPC |
| NAT Gateway | Outbound-only for private subnets | In Public Subnet A, with Elastic IP |
| Public Route Table | `0.0.0.0/0 → IGW` | Associated with both public subnets |
| Private Route Table | `0.0.0.0/0 → NAT Gateway` | Associated with both private subnets |

### Compute

| Instance | Role | Location | Public IP |
|---|---|---|---|
| `krish-web-server-a` | Web tier | Public Subnet A | Yes |
| `krish-web-server-b` | Web tier | Public Subnet B | Yes |
| `krish-app-server-a` | App tier | Private Subnet A | No |

All instances are Amazon Linux 2023 on `t2.micro` (free tier).

### Load Balancer

- **Application Load Balancer** (Layer 7)
- **Scheme**: Internet-facing
- **Targets**: Both web servers in a Target Group with health checks (interval 10s, healthy threshold 2)
- Routes only to healthy targets — used to demonstrate live AZ failover

### Security Groups

| Security Group | Inbound Rules | Reasoning |
|---|---|---|
| `krish-alb-sg` | HTTP (80) from `0.0.0.0/0` | Public entry point |
| `krish-web-sg` | HTTP (80) from `krish-alb-sg`; SSH (22) from my IP | Only ALB can reach web tier |
| `krish-app-sg` | SSH (22) + port 8080 from `krish-web-sg` | App tier only reachable from web tier |

Security groups are referenced by **SG ID** rather than IP. This is identity-based segmentation — no matter what IP addresses the servers happen to have, only members of the correct security group can reach them.

---

## Live Verification

The design was validated with real network tests, not just configuration screenshots:

### 1. Private tier is genuinely isolated

Attempt to reach the app server directly from my laptop's public internet:

```
$ curl -m 5 http://10.0.11.144:8080
curl: (28) Connection timed out after 5001 milliseconds
```

Timeout is the expected result. Three layers of "no" combine to block the request:
1. `10.0.11.x` is an RFC1918 private address — not routable on the internet
2. The private subnet has no Internet Gateway route
3. The app security group only accepts traffic from the web security group

### 2. Private tier still has outbound internet via NAT Gateway

From inside the app server (reached via the web server as bastion):

```
$ curl -m 5 -I https://www.google.com
HTTP/2 200
server: gws
...
```

The app server has **no public IP**, yet reached Google. Path taken:

```
app-server (10.0.11.144)
  → private route table (0.0.0.0/0 → NAT Gateway)
  → NAT Gateway (in public subnet, has Elastic IP)
  → Internet Gateway
  → internet
```

### 3. Load balancer distributes across AZs

Repeated requests to the ALB DNS name returned pages served by different backend instances — proving the load balancer is distributing traffic across both AZs.

### 4. Failover works

Stopping `krish-web-server-a`:
- Within ~20 seconds (2 × 10s health-check interval), the ALB marked it unhealthy
- All subsequent traffic was routed to `krish-web-server-b`
- Site remained available with zero interruption from the user's perspective

---

## Screenshots

Screenshots documenting each verified stage are in the [`screenshots/`](./screenshots) directory:

| # | Screenshot | Shows |
|---|---|---|
| 01 | VPC details | `krish-lab-vpc` with `10.0.0.0/16` |
| 02 | Subnets | All 4 subnets across 2 AZs |
| 03 | Public route table | `0.0.0.0/0 → IGW` |
| 04 | Private route table | `0.0.0.0/0 → NAT Gateway` |
| 05 | EC2 instances | 3 instances; app server has no public IP |
| 06 | Web tier SG | HTTP from ALB-SG, SSH from my IP |
| 07 | App tier SG | Only reachable from web-SG |
| 08 | ALB active | DNS name, internet-facing |
| 09 | Target group healthy | Both instances healthy |
| 10 | Load balancing in action | Different instances serving requests |
| 11 | Failover event | ALB routes around unhealthy instance |
| 12 | Private isolation proof | Timeout from public, success via bastion |

---

## What I Learned

- **AWS networking concepts map cleanly to enterprise routing.** A route table is a router. A default route (`0.0.0.0/0 → IGW`) is what makes a subnet "public." Everything follows from that.
- **Security Groups are stateful by default.** No need to add explicit return-traffic rules like on a traditional ACL.
- **The public/private split is defined by routing, not naming.** A subnet is "private" only because its route table has no path to an IGW. The label is documentation, not configuration.
- **NAT Gateways are expensive and easy to leave running.** Real cost discipline matters — I set up a `$5` billing alarm before creating any billable resources and deleted the NAT and released its EIP as soon as testing was complete.
- **Identity-based security scales better than IP-based.** Using security groups as sources means adding a new server just requires attaching it to the right SG — no ACL rewriting.

---

## Cost Discipline

This entire lab was completed for **under $2 AUD** in real AWS charges by following these rules:

1. Set up a CloudWatch billing alarm at $5 USD **before** creating any billable resource
2. Used free-tier instance types (`t2.micro`)
3. Kept the NAT Gateway running only for the ~2 hours of active testing
4. Deleted the NAT Gateway and released its Elastic IP the moment testing was verified
5. Left only free-tier resources running (VPC, subnets, route tables, SGs cost nothing)

---

## What's Next

Planned extensions for a follow-up project:
- Auto Scaling Group behind the ALB (self-healing at instance level as well as AZ level)
- CloudFront distribution in front of the ALB
- Terraform / CloudFormation Infrastructure-as-Code version of the same architecture
- CloudWatch dashboards and alarms

---

## About Me

I'm a final-year Bachelor of Information Technology student at Adelaide University, majoring in Networking & Cybersecurity. This project builds on my CCNP-level coursework in enterprise networking and is part of an AWS Cloud Practitioner (CLF-C02) study path.

- Portfolio: [krishbhadani.com](https://www.krishbhadani.com)
- LinkedIn: [linkedin.com/in/krish-bhadani-848598229](https://www.linkedin.com/in/krish-bhadani-848598229)
- GitHub: [github.com/krishbhadani](https://github.com/krishbhadani)

---

*Repository content is documentation and screenshots only — no credentials, keys, or account IDs are included.*
