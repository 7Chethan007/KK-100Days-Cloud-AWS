# Day 1 - AWS Security Group Creation

## Objective

As part of the Nautilus AWS migration initiative, create a Security Group in the **default VPC** with the following requirements:

- Security Group Name: `datacenter-sg`
- Description: `Security group for Nautilus App Servers`
- Inbound Rule:
  - HTTP (Port 80)
  - Source: `0.0.0.0/0`
- Inbound Rule:
  - SSH (Port 22)
  - Source: `0.0.0.0/0`

---

## Understanding the Task

A Security Group acts as a virtual firewall for AWS resources.

In this task, we needed to create a security group that would:

- Allow web traffic using HTTP on port 80.
- Allow remote administrative access using SSH on port 22.
- Accept connections from any IP address (`0.0.0.0/0`).

---

## Infrastructure Details

AWS credentials were provided through the KodeKloud lab environment.

### Retrieve Credentials

Login to the AWS client host and run:

```bash
showcreds
```

This displays:

- AWS Console URL
- Username
- Password
- Session Start Time
- Session End Time

---

## Steps Performed

### 1. Login to AWS Console

Used the provided AWS Console URL and credentials.

Selected Region:

```text
us-east-1 (N. Virginia)
```

> Note: The task explicitly required all resources to be created in the `us-east-1` region.

---

### 2. Navigate to Security Groups

From AWS Console:

```text
VPC → Security Groups
```

Click:

```text
Create Security Group
```

---

### 3. Configure Security Group Details

Entered:

| Field | Value |
|---------|---------|
| Security Group Name | datacenter-sg |
| Description | Security group for Nautilus App Servers |
| VPC | Default VPC |

---

### 4. Add HTTP Inbound Rule

Clicked:

```text
Add Rule
```

Configured:

| Setting | Value |
|-----------|-----------|
| Type | HTTP |
| Protocol | TCP |
| Port Range | 80 |
| Source | Anywhere IPv4 |
| CIDR | 0.0.0.0/0 |

### Why?

HTTP traffic uses port 80.

The CIDR range:

```text
0.0.0.0/0
```

means connections are allowed from any public IP address.

---

### 5. Add SSH Inbound Rule

Clicked:

```text
Add Rule
```

Configured:

| Setting | Value |
|-----------|-----------|
| Type | SSH |
| Protocol | TCP |
| Port Range | 22 |
| Source | Anywhere IPv4 |
| CIDR | 0.0.0.0/0 |

### Why?

SSH uses port 22 and allows remote administration of Linux servers.

---

### 6. Review Configuration

Final inbound rules:

| Type | Protocol | Port | Source |
|---------|---------|---------|---------|
| HTTP | TCP | 80 | 0.0.0.0/0 |
| SSH | TCP | 22 | 0.0.0.0/0 |

---

### 7. Create Security Group

Clicked:

```text
Create Security Group
```

AWS successfully created:

```text
datacenter-sg
```

---

## Validation

Verify the Security Group:

```text
Security Group Name:
datacenter-sg
```

Description:

```text
Security group for Nautilus App Servers
```

Inbound Rules:

```text
HTTP  TCP 80  0.0.0.0/0
SSH   TCP 22  0.0.0.0/0
```

---

## Key Learnings

### Security Groups

AWS Security Groups act as stateful virtual firewalls that control inbound and outbound traffic.

### CIDR Range

```text
0.0.0.0/0
```

represents all IPv4 addresses.

### HTTP Port

```text
80
```

used for web traffic.

### SSH Port

```text
22
```

used for secure remote server access.

### Default VPC

Every AWS account includes a default VPC, allowing resources to be launched without creating custom networking components.

---

## Outcome

Successfully created the AWS Security Group **datacenter-sg** in the **default VPC** within the **us-east-1** region and configured inbound access for:

- HTTP (Port 80)
- SSH (Port 22)

from any IPv4 address (`0.0.0.0/0`).
