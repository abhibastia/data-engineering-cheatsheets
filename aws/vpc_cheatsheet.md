# VPC Cheatsheet

---

## Core Concepts

- VPC = isolated private network in AWS; you own the IP address space
- Subnets are AZ-scoped — one subnet lives in one AZ only
- Public vs private is determined by the **route table**, not the subnet itself
- `local` route covers all internal VPC traffic and cannot be removed
- `0.0.0.0/0` = catch-all; used to route internet-bound traffic to IGW or NAT

---

## CIDR Quick Reference

| CIDR | Addresses | Use |
| --- | --- | --- |
| `/16` | 65,536 | VPC |
| `/24` | 256 | Subnet |
| `/32` | 1 | Single IP |

---

## Public vs Private Subnet

| | Public | Private |
| --- | --- | --- |
| Route table `0.0.0.0/0` target | IGW | NAT Gateway |
| Gets public IP | Yes (auto-assign) | No |
| Internet inbound | Yes | No |
| Internet outbound | Yes | Yes (via NAT) |

- ⚠️ "Public subnet" is not a subnet property — remove the `0.0.0.0/0 → IGW` route and it becomes private

---

## IGW vs NAT Gateway

| | IGW | NAT Gateway |
| --- | --- | --- |
| Inbound from internet | Yes | No |
| Cost | Free | ~$0.045/hr + data |
| Placement | Attached to VPC | Inside public subnet |

- ⚠️ NAT Gateway must be in a **public subnet** — placing it in private breaks all routing
- ⚠️ Delete NAT Gateway when done — forgotten NAT is one of the most common unexpected AWS bills

---

## Route Table Rules

```
Destination     Target
10.0.0.0/16     local          ← always present, cannot remove
0.0.0.0/0       igw-xxx        ← public subnet
0.0.0.0/0       nat-xxx        ← private subnet
```

More specific routes always win. `10.0.0.0/16` is more specific than `0.0.0.0/0`.

- ⚠️ A subnet with no explicit route table association falls back to the VPC's **main route table** by default

---

## Cleanup Order

1. Delete NAT Gateway
2. Wait for `deleted` state
3. Release Elastic IP
4. (VPC, subnets, IGW, route tables are free — safe to keep)

---

## CLI Quick Reference

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16                                                       # create VPC; returns VpcId
aws ec2 create-subnet --vpc-id $VPC --cidr-block 10.0.1.0/24 --availability-zone eu-central-1a   # subnet is private until a route table makes it public
aws ec2 modify-subnet-attribute --subnet-id $S --map-public-ip-on-launch                          # auto-assign public IPs to new instances in this subnet
aws ec2 create-internet-gateway                                                                    # create IGW; not attached until next command
aws ec2 attach-internet-gateway --internet-gateway-id $IGW --vpc-id $VPC                          # one VPC can have only one IGW
aws ec2 create-route-table --vpc-id $VPC                                                          # new route table; only has the local route initially
aws ec2 create-route --route-table-id $RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW  # default route to IGW; this makes the subnet public
aws ec2 associate-route-table --route-table-id $RT --subnet-id $S                                 # link route table to subnet
aws ec2 delete-nat-gateway --nat-gateway-id $NAT                                                  # delete NAT first; ~$0.045/hr while running
aws ec2 release-address --allocation-id $EIP                                                      # release Elastic IP after NAT deleted; billed while unattached
```
