# Auto Scaling Lab
**Author:** Alphonse Shema | **Region:** eu-north-1 (Stockholm) | **IaC:** AWS CloudFormation + GitSync

---

## Architecture Overview

```
                        Internet
                           |
                    [Internet Gateway]
                           |
         ┌─────────────────┴─────────────────┐
         │           VPC  10.0.0.0/16        │
         │                                   │
         │  AZ1 (eu-north-1a)  AZ2 (eu-north-1b)  │
         │  ┌──────────────┐  ┌──────────────┐    │
         │  │Public Subnet │  │Public Subnet │    │
         │  │ 10.0.1.0/24  │  │ 10.0.2.0/24  │    │
         │  │  [ALB Node]  │  │  [ALB Node]  │    │
         │  │ [NAT-GW]     │  │              │    │
         │  └──────┬───────┘  └──────┬───────┘    │
         │         │ (ALB routes)    │             │
         │  ┌──────▼───────┐  ┌──────▼───────┐    │
         │  │Private Subnet│  │Private Subnet│    │
         │  │ 10.0.3.0/24  │  │ 10.0.4.0/24  │    │
         │  │  [EC2 #1]    │  │  [EC2 #2]    │    │
         │  │  [EC2 #3]    │  │  [EC2 #4]    │    │
         │  │  (ASG)       │  │  (ASG)       │    │
         │  └──────────────┘  └──────────────┘    │
         └───────────────────────────────────────┘
```

---

## What This Template Deploys

| Resource | Count | Details |
|---|---|---|
| VPC | 1 | 10.0.0.0/16, DNS enabled |
| Internet Gateway | 1 | Attached to VPC |
| Public Subnets | 2 | One per AZ - for ALB |
| Private Subnets | 2 | One per AZ - for EC2 instances |
| Regional NAT Gateway | 1 | Single gateway, AWS manages AZ availability |
| Elastic IP | 1 | For NAT Gateway |
| Route Tables | 2 | 1 public, 1 private shared |
| ALB Security Group | 1 | HTTP from internet |
| EC2 Security Group | 1 | HTTP from ALB only |
| IAM Role + Instance Profile | 1 | SSM Session Manager access |
| Application Load Balancer | 1 | Internet-facing, round-robin |
| Target Group | 1 | Health checks every 30 seconds |
| ALB Listener | 1 | Port 80 HTTP |
| Launch Template | 1 | AMI, instance type, UserData, security group |
| Auto Scaling Group | 1 | Min 1, Desired 1, Max 4 |
| Scaling Policy | 1 | Target tracking - CPU threshold 30% |

---

## Repository Structure

```
autoscaling-lab/
├── autoscaling-lab.yaml    # Main CloudFormation template
└── README.md               # This file
```

---

## Deploy the Stack

### Option 1 - GitSync (Recommended)

1. Push `autoscaling-lab.yaml` to your GitHub repo on the `main` branch
2. Go to **CloudFormation -> Create stack -> Sync from Git**
3. Connect to your GitHub repo and select `autoscaling-lab.yaml`
4. Stack name: `autoscaling-lab-stack`
5. Tick the acknowledgement checkbox:
```
I acknowledge that AWS CloudFormation might create IAM resources with custom names.
```
6. Click **Submit**

### Option 2 - AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name autoscaling-lab-stack \
  --template-body file://autoscaling-lab.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region eu-north-1
```

> Full deployment takes approximately 5-10 minutes.

---

## Validation Tests

After the stack shows CREATE_COMPLETE, go to **CloudFormation -> Outputs** and copy the ALBUrl.

### Test 1 - Access the Application

Open the ALB URL in your browser. You should see:

```
Auto Scaling Lab
Instance ID:   i-xxxxxxxxxxxxxxxxx
Private IP:    10.0.x.x
Availability Zone: eu-north-1a or eu-north-1b
```

### Test 2 - Verify Load Balancing

Keep refreshing the ALB URL. The Instance ID and Private IP will change as the ALB routes requests to different instances (once you have more than one running).

### Test 3 - Connect via Session Manager

```
EC2 -> Instances -> select instance -> Connect -> Session Manager -> Connect
```

No SSH key needed.

### Test 4 - Trigger CPU Scale-Out

From a Session Manager terminal on any instance:

```bash
stress --cpu 4 --timeout 300
```

This runs for 5 minutes and spikes CPU above 30%. Watch in the console:

```
EC2 -> Auto Scaling Groups -> autoscaling-lab-asg -> Activity tab
```

New instances will appear within 1-2 minutes.

### Test 5 - Verify New Instances Join ALB

```
EC2 -> Target Groups -> autoscaling-lab-tg -> Targets tab
```

New instances appear here automatically and show as healthy after passing 2 health checks.

---

## Security Design

| Rule | ALB | EC2 Instances |
|---|---|---|
| Inbound HTTP port 80 | From internet | From ALB only |
| Inbound SSH port 22 | Blocked | Blocked |
| Direct internet access | Yes | No - via NAT only |
| Public IP | Yes | No |

---

## Auto Scaling Configuration

| Setting | Value | Why |
|---|---|---|
| Minimum capacity | 1 | Always at least one instance running |
| Desired capacity | 1 | Start with one instance |
| Maximum capacity | 4 | Cost cap - never more than 4 |
| Scale-out threshold | 30% CPU | Low threshold for easy demo |
| Scale-out cooldown | 60 seconds | Wait before adding another instance |
| Scale-in cooldown | 60 seconds | Wait before removing an instance |
| Health check type | ELB | ALB health checks control instance replacement |
| Health check grace period | 300 seconds | Give new instances time to install Apache |

---

## Regional NAT Gateway vs AZ-Scoped NAT Gateways

This lab uses ONE Regional NAT Gateway instead of two AZ-scoped ones.

| Feature | AZ-Scoped (VPC lab) | Regional (this lab) |
|---|---|---|
| Count | One per AZ (2 total) | One for whole region |
| High Availability | Manual | AWS manages automatically |
| Cross-AZ traffic | None | Possible |
| Cost | 2x hourly | 1x hourly |
| Route tables needed | One per private subnet | One shared |

Both private subnets point to the same single NAT Gateway. AWS handles failover.

---

## Cost

| Resource | Rate | Daily Cost |
|---|---|---|
| NAT Gateway x1 | $0.045 per hour | ~$1.08 |
| ALB | ~$0.022 per hour | ~$0.53 |
| t3.micro x1-4 | $0.0104 per hour each | ~$0.25-$1.00 |
| **Total** | | **~$1.86-$2.61 per day** |

> Delete the stack when not in use to stop all charges.

```
CloudFormation -> autoscaling-lab-stack -> Delete stack
```

---

## Author

**Alphonse Shema**
AWS Auto Scaling Lab
Region: eu-north-1 (Stockholm)
