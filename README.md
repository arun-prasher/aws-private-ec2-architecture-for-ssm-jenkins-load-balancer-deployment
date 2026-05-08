# AWS NAT + IGW + SSM Quick Notes  
## EC2 Instance for SSM / Jenkins Handling

### 1. Internet Gateway (IGW)

- IGW gives internet access to resources in a **public subnet**.
- A public subnet must have this route in its route table:

```text
0.0.0.0/0 → Internet Gateway
```

---

### 2. NAT Gateway

- NAT Gateway allows **private EC2 instances** to access the internet for outbound traffic.
- NAT Gateway must be created in a **public subnet**.
- NAT Gateway itself uses the **Internet Gateway**.

Traffic flow:

```text
Private EC2 → NAT Gateway → Internet Gateway → Internet
```

---

### 3. Private EC2 Instance

- Private EC2 should not have a public IP.
- This is more secure for backend services, workers, and Jenkins-handled servers.
- Private subnet route table should have:

```text
0.0.0.0/0 → NAT Gateway
```

---

### 4. SSM Requirements

For AWS Systems Manager Session Manager to work, the EC2 instance needs:

- IAM role attached to EC2 with this policy:

```text
AmazonSSMManagedInstanceCore
```

- Outbound HTTPS access:

```text
Port 443
```

- Either:
  - NAT Gateway access, or
  - SSM VPC endpoints

---

### 5. If SSM Fails

Most common causes:

- NAT Gateway is created in the wrong subnet
- Private subnet route table is not pointing to NAT Gateway
- Security group or NACL is blocking outbound HTTPS 443
- DNS is disabled in the VPC
- IAM role is missing or incorrect
- EC2 has no internet path through NAT or VPC endpoints

---

### 6. Correct Architecture

```text
PUBLIC SUBNET
- Internet Gateway route
- NAT Gateway
- Load Balancer, if required

PRIVATE SUBNET
- EC2 workers
- Django internal services
- SQS consumers
- Jenkins-handled backend servers
```

---

### 7. Best Practice for Production

- Keep only NAT Gateway and Load Balancer in the public subnet.
- Keep backend EC2 instances in private subnets.
- Use SSM Session Manager instead of SSH where possible.
- Avoid public IPs on backend EC2 instances.
- Allow outbound 443 from private EC2 instances.
- Attach the correct IAM role before testing SSM.

---

### Final Simple Rule

```text
Public subnet  → IGW
Private subnet → NAT Gateway
SSM needs      → IAM role + outbound 443 + NAT or VPC endpoints
```
