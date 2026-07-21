# Screenshots

Documentation screenshots of each verified stage of the AWS Cloud Networking Lab.

## File naming

Screenshots are numbered in the order they appear in the main README:

| File | What it shows |
|---|---|
| `01-vpc.png` | VPC `krish-lab-vpc` with CIDR `10.0.0.0/16` |
| `02-subnets.png` | All 4 subnets across 2 AZs |
| `03-public-route-table.png` | Public route table — `0.0.0.0/0 → IGW` |
| `04-private-route-table.png` | Private route table — `0.0.0.0/0 → NAT Gateway` |
| `05-instances.png` | EC2 instances — app server has no public IP |
| `06-web-sg.png` | Web tier Security Group inbound rules |
| `07-app-sg.png` | App tier Security Group inbound rules |
| `08-alb.png` | Application Load Balancer active |
| `09-target-group-healthy.png` | Target group with both instances healthy |
| `10-browser-a.png` | Browser showing ALB serving from instances a|
| `10-browser-b.png` | Browser showing ALB serving from instances b|
| `11-failover.png` | Failover event — one instance unhealthy |
| `12-private-isolation.png` | Terminal showing timeout from public + success via bastion |
