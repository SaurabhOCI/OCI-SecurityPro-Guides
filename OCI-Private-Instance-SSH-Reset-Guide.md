# OCI Private Instance SSH Key Reset Guide

## Overview

This guide provides a complete solution for resetting SSH keys on Oracle Cloud Infrastructure (OCI) compute instances located in **private subnets** when standard access methods (SSH, Bastion, Run Command, Serial Console) are unavailable or failing.

## Problem Statement

- Instance is in a private subnet (no public IP)
- SSH access is not working (key mismatch, lost key, or misconfiguration)
- OCI Bastion service showing "Invalid" status
- Instance Console Connection not working or no password set
- Run Command not responding

## Prerequisites

- OCI Console access with appropriate permissions
- A temporary jump host instance (can be created during this process)
- SSH key pair for authentication
- Basic knowledge of Linux commands and LVM

---

## Solution: Boot Volume Attachment Method

### Architecture

```
Jump Host (Public Subnet) 
    ↓
Boot Volume Detached from Private Instance
    ↓
Mount and Modify authorized_keys
    ↓
Reattach Boot Volume to Private Instance
```

---

## Step-by-Step Guide

### Phase 1: Create Jump Host

#### 1.1 Create Temporary Jump Host Instance

**In OCI Console:**

1. Navigate to **Compute → Instances → Create Instance**
2. Configure:
   - **Name:** `temp-jump-host`
   - **Compartment:** Same as your target instance
   - **Availability Domain:** Same as target instance
   - **Image:** Oracle Linux 8 (or any Linux)
   - **Shape:** VM.Standard.E2.1.Micro (free tier eligible)
3. **Networking:**
   - **VCN:** Same VCN as target instance
   - **Subnet:** **PUBLIC SUBNET** (critical!)
   - ✅ Check **"Assign a public IPv4 address"**
4. **SSH Keys:** Add your SSH public key
5. Click **Create**

#### 1.2 Connect to Jump Host

```bash
ssh -i <your-key>.key opc@<JUMP_HOST_PUBLIC_IP>
```

---

### Phase 2: Stop Target Instance and Detach Boot Volume

#### 2.1 Stop the Target Instance

1. Go to **Compute → Instances → [Your Private Instance]**
2. Click **Stop**
3. Wait for status: **Stopped**

#### 2.2 Detach Boot Volume

1. On the stopped instance page, click **Boot Volume** (left sidebar)
2. Click the **3-dot menu (⋮)** next to the boot volume
3. Select **Detach**
4. Confirm detachment
5. Wait for boot volume status: **Available**

---

### Phase 3: Attach Boot Volume to Jump Host

#### 3.1 Attach as Block Volume

1. Navigate to **Compute → Instances → temp-jump-host**
2. Click **Attached Block Volumes** (left sidebar)
3. Click **Attach Block Volume**
4. Configure:
   - **Volume Type:** Select **Boot Volume**
   - **Boot Volume:** Select your detached boot volume
   - **Device Path:** Leave as default
   - **Access Type:** **Read/Write**
   - **Attachment Type:** **Paravirtualized**
5. Click **Attach**
6. Wait for status: **Attached**

---

### Phase 4: Mount and Modify Boot Volume

#### 4.1 SSH to Jump Host and Identify Volume

```bash
ssh -i <your-key>.key opc@<JUMP_HOST_PUBLIC_IP>

# List all block devices
lsblk

# Check filesystem types
lsblk -f
```

You should see your attached boot volume (typically `/dev/sdb` with ~100GB size).

#### 4.2 Handle LVM Volume Groups

The boot volume uses LVM. If there's a naming conflict:

```bash
# Scan for volume groups
sudo vgscan

# You may see duplicate VG names warning
# Example: "VG name ocivolume is used by VGs <UUID1> and <UUID2>"

# List VG details to get UUIDs
sudo vgs -o vg_name,vg_uuid

# Activate using the UUID of the attached boot volume
sudo vgchange -ay --select vg_uuid=<ATTACHED_BOOT_VOLUME_UUID>

# If you see "device busy" errors, it's OK to proceed
```

#### 4.3 Mount the Boot Volume

```bash
# Create mount point
sudo mkdir -p /mnt/rescue

# Mount the root logical volume
# If activation created /dev/mapper/ocivolume-root:
sudo mount /dev/mapper/ocivolume-root /mnt/rescue

# OR if using a renamed VG:
sudo mount /dev/mapper/ocivolume-rescue-root /mnt/rescue

# Verify mount
ls /mnt/rescue
# Should show: bin boot dev etc home lib lib64 ...
```

#### 4.4 Add Your SSH Public Key

```bash
# Generate public key from your private key (if needed)
ssh-keygen -y -f ~/<your-key>.key > ~/<your-key>.pub

# View your public key
cat ~/<your-key>.pub

# Create .ssh directory if it doesn't exist
sudo mkdir -p /mnt/rescue/home/opc/.ssh

# REPLACE (not append) the authorized_keys file
sudo bash -c 'cat > /mnt/rescue/home/opc/.ssh/authorized_keys << EOF
<PASTE_YOUR_PUBLIC_KEY_HERE>
EOF'

# Set correct permissions
sudo chmod 700 /mnt/rescue/home/opc/.ssh
sudo chmod 600 /mnt/rescue/home/opc/.ssh/authorized_keys

# Set correct ownership (check UID first)
sudo cat /mnt/rescue/etc/passwd | grep opc
# Usually opc has UID 1000
sudo chown -R 1000:1000 /mnt/rescue/home/opc/.ssh

# Verify
sudo cat /mnt/rescue/home/opc/.ssh/authorized_keys
sudo ls -lan /mnt/rescue/home/opc/.ssh/
```

#### 4.5 Clear Cloud-Init SSH Cache (Critical!)

Cloud-init may overwrite SSH keys on boot. Clear its cache:

```bash
# Remove cloud-init SSH cache
sudo rm -f /mnt/rescue/var/lib/cloud/instance/sem/config_ssh*
sudo rm -f /mnt/rescue/var/lib/cloud/instances/*/sem/config_ssh*
```

#### 4.6 Set a Password (Optional but Recommended)

For future serial console access:

```bash
# Chroot into the system
sudo mount --bind /dev /mnt/rescue/dev
sudo mount --bind /proc /mnt/rescue/proc
sudo mount --bind /sys /mnt/rescue/sys
sudo chroot /mnt/rescue

# Set password
passwd opc

# Exit chroot
exit

# Unmount bind mounts
sudo umount /mnt/rescue/sys
sudo umount /mnt/rescue/proc
sudo umount /mnt/rescue/dev
```

#### 4.7 Unmount and Deactivate

```bash
# Unmount
sudo umount /mnt/rescue

# Deactivate volume group (use UUID if needed)
sudo vgchange -an --select vg_uuid=<ATTACHED_BOOT_VOLUME_UUID>

# Or simply:
sudo vgchange -an ocivolume
```

---

### Phase 5: Reattach and Test

#### 5.1 Detach from Jump Host

**In OCI Console:**

1. Go to **Compute → Instances → temp-jump-host**
2. Click **Attached Block Volumes**
3. Find the boot volume
4. Click **3-dot menu → Detach**
5. Confirm and wait for detachment

#### 5.2 Reattach to Private Instance

1. Go to **Compute → Instances → [Your Private Instance]**
2. Click **Boot Volume** (left sidebar)
3. Click **Attach Boot Volume**
4. Select your boot volume
5. Click **Attach**
6. Wait for attachment to complete

#### 5.3 Start Instance

1. Click **Start** button
2. Wait **3-5 minutes** for the instance to fully boot

#### 5.4 Test SSH Access

From the jump host:

```bash
ssh -i <your-key>.key opc@<PRIVATE_INSTANCE_PRIVATE_IP>
```

✅ **Success!** You should now have SSH access.

---

### Phase 6: Clean Up

#### 6.1 Terminate Jump Host (Optional)

Once you have access to your private instance:

1. Go to **Compute → Instances → temp-jump-host**
2. Click **Terminate**
3. Confirm termination

**Note:** Keep the jump host if you frequently need to access instances in private subnets.

---

## Alternative Methods

### Method 1: Serial Console (If You Have a Password)

If you previously set a password:

1. Go to your instance in OCI Console
2. Click **Console Connection** (left sidebar)
3. Click **"Connect with Cloud Shell"**
4. Login with username and password
5. Manually fix SSH configuration

### Method 2: Configure SSH Keys During Instance Creation Only

**Important:** OCI does not allow adding or modifying SSH keys on existing instances through the Console UI. The SSH keys section is read-only after instance creation.

For **new instances**, you can configure SSH keys during creation:

1. During instance creation, in the **"Add SSH keys"** section
2. Choose one of:
   - **"Generate a key pair for me"** - OCI generates and downloads keys
   - **"Upload public key files (.pub)"** - Upload your .pub file
   - **"Paste public keys"** - Paste public key content
3. Complete instance creation

**For existing instances:** Use the boot volume attachment method described in this guide (Phase 2-5).

---

## Troubleshooting

### Issue 1: Duplicate Volume Group Names

**Problem:** 
```
WARNING: VG name ocivolume is used by VGs <UUID1> and <UUID2>
Multiple VGs found with the same name: skipping ocivolume
```

**Solution:**
Use the UUID-based activation:
```bash
sudo vgchange -ay --select vg_uuid=<CORRECT_UUID>
```

To find the correct UUID:
```bash
sudo vgs -o vg_name,vg_uuid
sudo pvs -o pv_name,vg_name,vg_uuid
```

---

### Issue 2: "Device or Resource Busy" When Mounting

**Problem:**
```
device-mapper: create ioctl on ocivolume-root failed: Device or resource busy
```

**Solution:**
This warning can often be ignored. Try mounting anyway:
```bash
sudo mount /dev/mapper/ocivolume-root /mnt/rescue
```

If mount still fails:
```bash
# Check what's using the device
sudo dmsetup info ocivolume-root
sudo lsof | grep ocivolume

# Try using the dm device directly
ls -la /dev/mapper/
sudo mount /dev/dm-X /mnt/rescue  # Replace X with correct number
```

---

### Issue 3: System Boots to Dracut Emergency Shell

**Problem:**
Instance boots but drops to `dracut:/#` emergency shell with errors like:
```
Warning: /dev/mapper/ocivolume-root does not exist
```

**Cause:** Volume group was renamed but boot configuration still references old name.

**Solution:**

From dracut emergency shell:
```bash
# Activate volume group
lvm vgscan
lvm vgchange -ay

# Check VG name
lvm lvs

# If VG is named something other than "ocivolume", rename it:
lvm vgrename <current-name> ocivolume

# Mount manually
mount /dev/mapper/ocivolume-root /sysroot

# Chroot and fix permanently
mount --bind /dev /sysroot/dev
mount --bind /proc /sysroot/proc
mount --bind /sys /sysroot/sys
chroot /sysroot

# Regenerate initramfs
dracut -f

# Exit and reboot
exit
umount /sysroot/sys /sysroot/proc /sysroot/dev /sysroot
reboot
```

---

### Issue 4: SSH Key Rejected Despite Correct Configuration

**Problem:**
- authorized_keys has correct key
- Permissions are correct (700/.ssh, 600/authorized_keys)
- SSH still returns "Permission denied (publickey)"

**Possible Causes & Solutions:**

#### A. Cloud-Init Overwrites Keys on Boot

**Check:**
```bash
sudo cat /mnt/rescue/var/lib/cloud/instance/obj.pkl | strings | grep ssh-rsa
```

**Solution:**
```bash
# Clear cloud-init cache
sudo rm -f /mnt/rescue/var/lib/cloud/instance/sem/config_ssh*
sudo rm -f /mnt/rescue/var/lib/cloud/instances/*/sem/config_ssh*
```

#### B. Wrong Key in Instance Metadata (from Creation)

**Note:** OCI does not allow modifying SSH keys on existing instances through the Console.

**Solution:**
The instance was created with a different SSH key stored in its initial configuration. You must:
1. Use the boot volume attachment method (Phases 2-5)
2. Clear cloud-init cache as shown above
3. Replace authorized_keys with your key

Or create a new instance with the correct key if the instance is not critical.

#### C. SELinux Context Issues

**Check:**
```bash
sudo ls -laZ /mnt/rescue/home/opc/.ssh/
```

**Solution:**
```bash
# Restore SELinux contexts
sudo chroot /mnt/rescue
restorecon -Rv /home/opc/.ssh
exit
```

#### D. Verify Key Fingerprints Match

```bash
# Server-side (mounted volume)
sudo ssh-keygen -lf /mnt/rescue/home/opc/.ssh/authorized_keys

# Client-side (your key)
ssh-keygen -lf <your-key>.key

# Fingerprints should match exactly
```

---

### Issue 5: Serial Console Connection Failures

**Problem:** 
```
Unable to negotiate with UNKNOWN port 65535: no matching host key type found. Their offer: ssh-rsa
```

**Cause:** Modern OpenSSH (8.8+) disabled ssh-rsa by default.

**Solution:**
Add ssh-rsa compatibility options:
```bash
ssh -i <key>.key \
  -o HostKeyAlgorithms=+ssh-rsa \
  -o PubkeyAcceptedKeyTypes=+ssh-rsa \
  -o ProxyCommand='ssh -i <key>.key -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa -W %h:%p -p 443 ocid1.instanceconsoleconnection.oc1...@instance-console.<region>.oci.oraclecloud.com' \
  ocid1.instance.oc1...
```

**Or use Cloud Shell instead:**
1. In OCI Console, go to Console Connection
2. Click "Connect with Cloud Shell"
3. No ssh-rsa configuration needed

---

### Issue 6: Bastion Plugin Shows "Invalid"

**Problem:**
- Bastion plugin status: Invalid
- Error: "Stale plugin data, generated more than 500 min back"

**Cause:**
- Oracle Cloud Agent cannot communicate with OCI control plane
- Usually due to missing Service Gateway route

**Solution:**

#### A. Check Network Configuration
1. Navigate to **VCN → Route Tables** for your private subnet
2. Verify route exists:
   - **Destination:** All [region] Services in Oracle Services Network
   - **Target:** Service Gateway
3. If missing, add the route

#### B. Reboot Instance
After fixing network:
1. Reboot instance
2. Wait 5-10 minutes
3. Check Bastion plugin status (should become "Running")

#### C. Restart Oracle Cloud Agent
If reboot doesn't work:
```bash
# SSH to instance (via jump host)
sudo systemctl restart oracle-cloud-agent
sudo systemctl status oracle-cloud-agent
```

---

### Issue 7: iSCSI Commands Not Working (Paravirtualized Volumes)

**Problem:**
```
iscsiadm: cannot make connection to 169.254.2.2: Connection refused
```

**Cause:**
Boot volume is attached as **Paravirtualized** type (not iSCSI).

**Solution:**
No iSCSI commands needed! The disk is automatically available:
```bash
lsblk
# The attached boot volume appears as /dev/sdb (or similar)
# Just mount it directly after activating LVM
```

---

### Issue 8: Permission Denied with Correct Key

**Problem:**
SSH verbose output shows key being sent but server rejects it:
```
debug1: Trying private key: <key>
debug3: sign_and_send_pubkey: RSA SHA256:...
debug1: Authentications that can continue: publickey
```

**Root Cause Analysis:**

Check server-side SSH logs:
```bash
sudo cat /mnt/rescue/var/log/secure | grep sshd | tail -50
```

Look for errors like:
- `Authentication refused: bad ownership or modes for directory /home/opc`
- `Authentication refused: bad ownership or modes for file /home/opc/.ssh/authorized_keys`

**Solution:**
```bash
# Fix home directory permissions
sudo chmod 755 /mnt/rescue/home/opc
sudo chown 1000:1000 /mnt/rescue/home/opc

# Fix .ssh directory
sudo chmod 700 /mnt/rescue/home/opc/.ssh
sudo chown 1000:1000 /mnt/rescue/home/opc/.ssh

# Fix authorized_keys
sudo chmod 600 /mnt/rescue/home/opc/.ssh/authorized_keys
sudo chown 1000:1000 /mnt/rescue/home/opc/.ssh/authorized_keys

# Verify
sudo ls -lan /mnt/rescue/home/opc/.ssh/
```

---

### Issue 9: Cannot Edit SSH Keys in OCI Console

**Problem:**
User tries to add SSH keys to an existing instance but cannot find the option in Edit interface.

**Explanation:**
This is **not a bug** - OCI does not allow modifying SSH keys on existing instances through the Console UI. The SSH keys shown in the instance details are read-only and reflect what was configured during instance creation.

**Why This Limitation Exists:**
- Security: Prevents unauthorized key injection
- SSH keys are managed at the OS level, not OCI infrastructure level
- Cloud-init only runs during first boot

**Solution:**
For existing instances, you must use one of these methods:
1. **Boot volume attachment method** (Phases 2-5 in this guide) - Works when you have no access
2. **SSH access** - If you have access: `echo "key" >> ~/.ssh/authorized_keys`
3. **Serial console** - If you have a password set, login and modify authorized_keys
4. **Bastion service** - If properly configured, use Bastion to access and modify

**Prevention:**
- Always configure correct SSH keys during instance creation
- Store SSH keys in a secure password manager
- Test SSH access immediately after instance creation
- Set a password for serial console access as a backup

---

## Best Practices

### 1. Always Keep Multiple Access Methods

- **SSH Keys:** Store in secure password manager
- **Serial Console:** Set a password for emergency access
- **Bastion:** Ensure Service Gateway is configured
- **Jump Host:** Consider keeping a permanent jump host for private subnets

### 2. Test Serial Console Access Periodically

```bash
# Set password via SSH while you have access
sudo passwd opc

# Test serial console connection
# (via OCI Console → Console Connection → Connect with Cloud Shell)
```

### 3. Document Your SSH Keys

Keep a record of which SSH key fingerprints are used for which instances:

```bash
# Generate fingerprint
ssh-keygen -lf <your-key>.key
```

### 4. Configure SSH Keys During Instance Creation

**Critical:** OCI does not allow adding/modifying SSH keys on existing instances via Console.

When launching instances, **always** configure SSH keys properly:
- Use the instance creation wizard SSH keys section
- Choose "Paste public keys" or "Upload public key files"
- Save your private key securely
- **You cannot change these keys later via Console UI**

For existing instances without access, use the boot volume attachment method.

### 5. Configure Proper Network Access

For instances in private subnets:
- Ensure VCN has Service Gateway
- Add route to Service Gateway in private subnet route table
- Verify Security Lists allow required traffic

### 6. Enable Oracle Cloud Agent

Ensure Oracle Cloud Agent is:
- Installed (comes with Oracle Linux images)
- Running
- Has network access to OCI control plane (via Service Gateway)

### 7. Backup Before Major Changes

Before modifying boot volumes:
```bash
# Create boot volume backup
OCI Console → Boot Volumes → Create Backup
```

---

## Security Considerations

### 1. Jump Host Security

- Use strong SSH keys
- Enable firewall rules
- Update regularly: `sudo yum update`
- Consider using OCI Bastion service instead of persistent jump host
- Use Security Lists to restrict jump host access to your IP only

### 2. SSH Key Management

- Use strong key sizes (RSA 2048+ or Ed25519)
- Protect private keys with passphrase
- Never commit private keys to version control
- Rotate keys periodically

### 3. Boot Volume Handling

- Detached boot volumes contain all instance data
- Ensure only authorized users can attach volumes
- Use IAM policies to control volume operations
- Delete jump host after use if it's temporary

### 4. Audit Trail

All actions are logged in OCI Audit service:
- Volume attachments/detachments
- Instance starts/stops
- Console connections

Review audit logs for any unauthorized access attempts.

---

## Common Mistakes to Avoid

### ❌ Don't: Append Keys Repeatedly

```bash
# This creates duplicate keys:
echo "<key>" >> authorized_keys
echo "<key>" >> authorized_keys  # Duplicate!
```

### ✅ Do: Replace the File

```bash
# Use > to replace, not >>
cat > authorized_keys << EOF
<key>
EOF
```

### ❌ Don't: Ignore Permissions

Wrong permissions will cause SSH to reject keys:
```bash
chmod 777 .ssh  # Wrong!
chmod 644 authorized_keys  # Wrong!
```

### ✅ Do: Use Correct Permissions

```bash
chmod 700 .ssh
chmod 600 authorized_keys
```

### ❌ Don't: Rename Volume Groups Without Updating Boot Config

If you rename a VG, the system won't boot.

### ✅ Do: Keep Original VG Name or Update Initramfs

```bash
# If you renamed VG, regenerate initramfs:
chroot /mnt/rescue
dracut -f
exit
```

### ❌ Don't: Forget to Clear Cloud-Init Cache

Cloud-init will overwrite your manual SSH key changes.

### ✅ Do: Clear Cache or Update Metadata

```bash
# Clear cache
rm -f /mnt/rescue/var/lib/cloud/instance/sem/config_ssh*

# OR add key to instance metadata via OCI Console
```

---

## Quick Reference Commands

### Generate SSH Key Pair
```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/oci-key -C "oci-instance-key"
```

### Extract Public Key from Private Key
```bash
ssh-keygen -y -f private-key.key > public-key.pub
```

### Check Key Fingerprint
```bash
ssh-keygen -lf key-file
```

### Mount LVM Volume
```bash
sudo vgscan
sudo vgchange -ay <vg-name>
sudo mount /dev/<vg-name>/<lv-name> /mnt/rescue
```

### Fix SSH Permissions (Standard)
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Test SSH with Verbose Output
```bash
ssh -vvv -i key.pem user@host
```

---

## Useful OCI CLI Commands

### List Instances
```bash
oci compute instance list --compartment-id <compartment-ocid>
```

### Stop Instance
```bash
oci compute instance action --instance-id <instance-ocid> --action STOP
```

### Start Instance
```bash
oci compute instance action --instance-id <instance-ocid> --action START
```

### List Boot Volumes
```bash
oci bv boot-volume list --availability-domain <ad> --compartment-id <compartment-ocid>
```

### Detach Boot Volume
```bash
oci compute boot-volume-attachment detach --boot-volume-attachment-id <attachment-ocid>
```

---

## Additional Resources

### Official Documentation
- [OCI Compute Documentation](https://docs.oracle.com/en-us/iaas/Content/Compute/home.htm)
- [OCI Bastion Service](https://docs.oracle.com/en-us/iaas/Content/Bastion/home.htm)
- [Serial Console Connections](https://docs.oracle.com/en-us/iaas/Content/Compute/References/serialconsole.htm)
- [Oracle Cloud Agent](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/manage-plugins.htm)

### Community Resources
- [OCI Community Forums](https://community.oracle.com/tech/apps-infra/categories/oci-compute)
- [OCI GitHub Examples](https://github.com/oracle/oci-cli)

---

## Frequently Asked Questions

### Q: Can I add or change SSH keys on an existing instance through OCI Console?

**A:** **No.** This is a common misconception. OCI does not provide a UI option to add or modify SSH keys on existing instances. The SSH keys section in instance details is read-only.

**Why?** SSH keys are configured at the OS level during first boot (via cloud-init), not managed ongoing by OCI infrastructure. For security reasons, OCI doesn't inject keys into running or stopped instances.

**Solution:** Use the boot volume attachment method described in this guide to manually modify the `authorized_keys` file.

### Q: Can I do this for Windows instances?

**A:** The general process is similar, but:
- Windows uses `authorized_keys` in `C:\Users\<username>\.ssh\` or `C:\ProgramData\ssh\administrators_authorized_keys`
- File permissions work differently (use `icacls` to set)
- May need to enable SSH server in Windows (OpenSSH Server feature)

### Q: How long does this process take?

**A:** 
- Jump host creation: 3-5 minutes
- Volume detach/attach: 1-2 minutes each
- Instance boot: 3-5 minutes
- Manual modifications: 5-10 minutes
- **Total: ~20-30 minutes**

### Q: Will I lose data during this process?

**A:** No, this is a non-destructive process. You're only mounting the boot volume read-write and modifying SSH configuration files. However, always backup critical data before maintenance.

### Q: Can I use this for instances with encrypted boot volumes?

**A:** Yes, but you need appropriate KMS key access. OCI handles encryption/decryption transparently when you attach the volume.

### Q: What if I don't know which SSH key was originally used?

**A:** That's fine! This guide helps you add a NEW key that you control. The old key will still work too (unless you replace the authorized_keys file entirely).

### Q: Can I automate this with Terraform or OCI CLI?

**A:** Yes! All OCI Console actions can be done via CLI or Terraform:
- Instance stop/start: `oci compute instance action`
- Volume detach/attach: `oci compute boot-volume-attachment`
- The mounting and file modification still needs to be done manually or via automation scripts

### Q: My instance is in a different region. Does this work?

**A:** Yes, this process works in all OCI regions. Just ensure the jump host is in the same region as the target instance.

---

## Version History

- **v1.0** (2025-11-05): Initial comprehensive guide
  - Boot volume attachment method
  - Cloud-init troubleshooting
  - LVM handling for duplicate VG names
  - Serial console SSH compatibility
  - Complete troubleshooting section

---

## Contributing

Found an issue or have an improvement? Please contribute:
1. Fork the repository
2. Create a feature branch
3. Submit a pull request

---

## License

This guide is provided as-is for educational purposes. Always test in non-production environments first.

---

## Support

For OCI-specific issues, contact Oracle Cloud Support or use the [OCI Community Forums](https://community.oracle.com/).

For this guide, please open an issue in the repository.

---

**Created by:** DevOps/Cloud Engineering Community  
**Last Updated:** November 5, 2025  
**Tested On:** Oracle Linux 8, OCI Compute (ARM & x86)
