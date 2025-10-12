# OCI Zero Trust Packet Routing (ZPR) - Practice Lab

## Overview

This repository demonstrates how to implement **Oracle Cloud Infrastructure (OCI) Zero Trust Packet Routing (ZPR)** to enforce granular, intent-based security policies without modifying existing network configurations or security lists.

### What is OCI ZPR?

OCI ZPR operates on the **"Zero Trust" principle** - no user or device, whether inside or outside the network, is inherently trusted. Instead of relying solely on IP addresses and network topology, ZPR introduces **intent-based security policies** using security attributes.

## Use Case Scenario

**Initial Setup:**
- Two compute instances (VM-01 and VM-02) in a public subnet
- Both instances are accessible via SSH from the internet

**Security Goal:**
- ✅ Allow SSH access to VM-01 from the internet
- ✅ Allow SSH access to VM-02 only from VM-01
- ❌ Block direct SSH access to VM-02 from the internet

**Solution:** Implement OCI ZPR without changing existing security lists!

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  OCI Region                      │
│  ┌───────────────────────────────────────────┐  │
│  │ VCN (10.0.0.0/16)                         │  │
│  │  ┌─────────────────────────────────────┐  │  │
│  │  │ Public Subnet                       │  │  │
│  │  │                                     │  │  │
│  │  │  ┌──────┐         ┌──────┐        │  │  │
│  │  │  │VM-01 │◄────────┤VM-02 │        │  │  │
│  │  │  └───▲──┘   SSH   └──────┘        │  │  │
│  │  │      │                             │  │  │
│  │  │      │ SSH (Allowed)               │  │  │
│  │  └──────┼─────────────────────────────┘  │  │
│  │         │                                 │  │
│  │    ┌────▼────┐                           │  │
│  │    │   IGW   │                           │  │
│  └────┴─────────┴───────────────────────────┘  │
│              │                                  │
└──────────────┼──────────────────────────────────┘
               │
         ┌─────▼──────┐
         │  Internet  │
         │   (User)   │
         └────────────┘
```

## Implementation Steps

### Step 1: Setting up the Network Infrastructure

1. **Create VCN and Public Subnet**
   - VCN CIDR: `10.0.0.0/16`
   - Public Subnet with Internet Gateway

2. **Configure Route Table**
   - Add route rule: `0.0.0.0/0` → Internet Gateway

3. **Configure Security List**
   - Ingress Rule: Allow TCP port 22 from `0.0.0.0/0`

### Step 2: Creating Compute Instances

1. Launch two compute instances:
   - **VM-01** in the public subnet
   - **VM-02** in the public subnet

2. Verify both instances are SSH accessible from your local machine

### Step 3: Setup OCI ZPR

#### 3.1 Enable OCI ZPR

1. Navigate to: **Identity & Security** → **Zero Trust Packet Routing**
2. Click **Enable ZPR**
3. Confirm by clicking **Enable ZPR** again

#### 3.2 Create Security Attribute Namespace

1. Click **Security Attribute Namespace**
2. Click **Create Security Attribute Namespace**
3. Name: `ZPRDEMO-01`

#### 3.3 Create Security Attributes

Create the following security attributes in the `ZPRDEMO-01` namespace:

**For Compute Instances:**
- **Attribute Name:** `app`
- **Type:** Static
- **Values:**
  - `vm01` (for VM-01)
  - `vm02` (for VM-02)

**For VCN Network:**
- **Attribute Name:** `network`
- **Type:** Static
- **Values:**
  - `vcn01` (for your VCN)

#### 3.4 Create ZPR Policies

Navigate to **Policies** → **Create policy** → Select **Manual Option**

**Copy and paste these policy statements:**

```
in ZPRDEMO-01.network:vcn01 VCN allow ZPRDEMO-01.app:vm01 endpoints to connect to ZPRDEMO-01.app:vm02 endpoints with protocol='tcp/22'

in ZPRDEMO-01.network:vcn01 VCN allow '0.0.0.0/0' to connect to ZPRDEMO-01.app:vm01 endpoints with protocol='tcp/22'
```

**Policy Explanation:**
- **Policy 1:** Allows VM-01 to establish SSH connections to VM-02 within the VCN
- **Policy 2:** Allows SSH connections from the internet (or your public IP) to VM-01

> **Security Tip:** Replace `0.0.0.0/0` with your specific public IP CIDR for enhanced security

#### 3.5 Add Resources to ZPR

1. Click **Protected Resources**
2. Click **Add security attribute to resource**
3. Apply security attributes to resources:

| Resource | Security Attribute | Value |
|----------|-------------------|-------|
| VM-01    | app               | vm01  |
| VM-02    | app               | vm02  |
| VCN      | network           | vcn01 |

### Step 4: Verifying the ZPR Setup

#### Test 1: Direct SSH to VM-01 ✅

```bash
ssh -i <path_to_vm01_private_key> opc@<public_ip_of_vm01>
```

**Expected Result:** Connection succeeds (allowed by ZPR policy)

#### Test 2: Direct SSH to VM-02 ❌

```bash
ssh -i <path_to_vm02_private_key> opc@<public_ip_of_vm02>
```

**Expected Result:** Connection fails with "Connection refused" (blocked by ZPR policy)

#### Test 3: SSH to VM-02 via VM-01 ✅

**Step 3a:** Copy VM-02's private key to VM-01

```bash
scp -i <path_to_vm01_private_key> <path_to_vm02_private_key_file> opc@<public_ip_of_vm01>:/home/opc
```

**Step 3b:** SSH to VM-01

```bash
ssh -i <path_to_vm01_private_key> opc@<public_ip_of_vm01>
```

**Step 3c:** From VM-01, SSH to VM-02 using its private IP

```bash
ssh -i /home/opc/<vm02_private_key_filename> opc@<private_ip_of_vm02>
```

**Expected Result:** Connection succeeds (allowed by ZPR policy)

## Key Concepts

### Zero Trust Principles

- **No implicit trust:** All connections must be explicitly authorized
- **Principle of least privilege:** Grant only necessary access
- **Intent-based security:** Policies based on resource attributes, not just IP addresses

### How ZPR Works

ZPR adds an additional security layer on top of existing network controls:

```
Connection Allowed = Network Rules (Security Lists/NSG) AND ZPR Policies
```

Even if security lists allow traffic, ZPR policies must also permit it.

## Benefits

✅ **Prevent misconfigurations** in large network setups  
✅ **Granular access control** without modifying network rules  
✅ **Intent-based policies** that are easier to understand and maintain  
✅ **Defense in depth** with multiple security layers  
✅ **Compliance** with zero trust security frameworks

## Cleanup

To remove the ZPR configuration:

1. Remove security attributes from resources
2. Delete ZPR policies
3. Delete security attributes
4. Delete security attribute namespace
5. (Optional) Disable ZPR service

## References

- [Official OCI ZPR Documentation](https://docs.oracle.com/en-us/iaas/Content/zero-trust-packet-routing/home.htm)
- [First Principles: Zero Trust Packet Routing](https://blogs.oracle.com/cloud-infrastructure/post/first-principles-zero-trust-packet-routing)
- [Announcing GA of OCI ZPR Service](https://blogs.oracle.com/cloud-infrastructure/post/ga-zero-trust-packet-routing)

## Troubleshooting

**Issue:** Cannot connect to VM-01 after enabling ZPR

**Solution:** Verify the second ZPR policy allows your IP address and check that security attributes are correctly applied

**Issue:** Can still connect to VM-02 directly

**Solution:** Ensure VM-02 has the correct security attribute (`app:vm02`) applied and ZPR is enabled

## Experimentation

Feel free to experiment with different policy statements to understand how ZPR works:

- Try allowing specific ports
- Test with different protocol restrictions
- Create additional security attributes for more complex scenarios

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests with improvements, additional examples, or corrections.

## License

This project is licensed under the MIT License - see the LICENSE file for details.

---

**Note:** Always follow your organization's security policies and best practices when implementing cloud infrastructure.
