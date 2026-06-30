# EC2 Cheatsheet

---

## Core Concepts

- EC2 = virtual server on demand; billed per second (Linux) or per hour (Windows) while running
- **AMI** = OS + software blueprint; region-specific
- **Instance type** = CPU/memory profile; pick based on workload
- **Key pair** = SSH auth; private key downloaded once, never recoverable — lose it, lose access
- **Security group** = stateful firewall; whitelist only; attached to instance, not subnet
- **User data** = script that runs on first boot only; failures are silent

---

## Instance Families

| Family | Use for |
| --- | --- |
| t3/t4g | Dev/test, burstable workloads |
| m5/m6i | General pipeline workers, Airflow |
| c5/c6i | Heavy data transformation, Spark |
| r5/r6i | Large in-memory operations |
| i3/i4i | NVMe-heavy Spark shuffle |

---

## AMI Usernames

| AMI | SSH username |
| --- | --- |
| Amazon Linux 2 | `ec2-user` |
| Ubuntu | `ubuntu` |
| Windows | `Administrator` (RDP) |

- ⚠️ Wrong username = `Permission denied (publickey)` — looks like an SSH key error but isn't

---

## Security Groups

- **Stateful** — allow inbound TCP 80, response is automatically allowed out
- VPC-specific — cannot reuse across VPCs
- Changes apply immediately to all attached instances
- ⚠️ No explicit allow = implicit deny — nothing gets through unless you added a rule
- Use SG-to-SG references for internal traffic instead of CIDR ranges

---

## Key Pair Rules

- Downloaded once — AWS does not store the private key
- Region-specific — key in `eu-central-1` cannot be used in `us-east-1`
- `chmod 400` is mandatory — SSH refuses keys others can read
- ⚠️ Lost key = terminate instance and relaunch with new key

---

## Instance Lifecycle

```
pending → running → stopping → stopped → terminated
```

- Stop = halts instance; EBS persists; no compute charge (storage still billed)
- Start after stop = new public IP (unless Elastic IP attached)
- Terminate = permanent; root EBS deleted by default
- ⚠️ Termination cannot be undone

---

## User Data

- Runs once, at first boot only
- Failures are silent — instance passes health checks regardless
- Package manager must match AMI: `yum` for Amazon Linux, `apt` for Ubuntu
- Debug: `aws ec2 get-console-output --instance-id $ID --query "Output" --output text`

---

## CLI

```bash
aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
  --key-name my-key --security-group-ids $SG --subnet-id $SUBNET \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=my-ec2}]' \
  --query "Instances[0].InstanceId" --output text                      # launch and return instance ID

aws ec2 wait instance-running --instance-ids $INSTANCE_ID             # block until instance reaches running state

aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].{ID:InstanceId,IP:PublicIpAddress}" \
  --output table                                                        # list running instances with public IPs

aws ec2 stop-instances --instance-ids $INSTANCE_ID                    # halt; EBS persists; no compute charge
aws ec2 terminate-instances --instance-ids $INSTANCE_ID               # permanent; root EBS deleted by default

ssh -i ~/my-key.pem ubuntu@<PUBLIC-IP>                                 # adjust username per AMI (ec2-user for Amazon Linux)
```
