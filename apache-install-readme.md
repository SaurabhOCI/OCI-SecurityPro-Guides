# Apache Web Server Installation Guide

This guide provides step-by-step instructions for installing and configuring Apache Web Server on a compute instance via SSH.

## Prerequisites

- Access to a compute instance via SSH
- Sudo privileges on the instance
- Basic familiarity with command-line interface

## Installation Steps

### 1. Connect to Your Compute Instance

Connect to your compute instance using SSH:

```bash
ssh username@your-instance-ip
```

### 2. Install Apache Web Server

Update your package manager and install Apache:

```bash
sudo yum install httpd -y
```

### 3. Enable and Start Apache Service

Enable Apache to start automatically on boot and start the service:

```bash
sudo apachectl start
sudo systemctl enable httpd
```

### 4. Verify Apache Configuration

Run a configuration test to ensure everything is set up correctly:

```bash
sudo apachectl configtest
```

You should see a "Syntax OK" message if the configuration is correct.

### 5. Configure Firewall Rules

Create firewall rules to allow HTTP traffic through port 80:

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --reload
```

These commands will:
- Add port 80 to the allowed ports permanently
- Reload the firewall to apply the changes

### 6. Create a Test Index Page

Create a simple HTML index file to verify your web server is working:

```bash
sudo bash -c 'echo "You are visiting Web Server 1" >> /var/www/html/index.html'
```

### 7. Exit SSH Connection

When finished, exit the SSH session:

```bash
exit
```

## Verification

To verify your web server is running correctly:

1. Open a web browser
2. Navigate to your instance's public IP address: `http://your-instance-ip`
3. You should see the message: "You are visiting Web Server 1"

## Troubleshooting

If you encounter issues:

- Check Apache status: `sudo systemctl status httpd`
- View Apache error logs: `sudo tail -f /var/log/httpd/error_log`
- Verify firewall rules: `sudo firewall-cmd --list-all`
- Ensure SELinux isn't blocking: `sudo getenforce`

## Additional Resources

- [Apache HTTP Server Documentation](https://httpd.apache.org/docs/)
- [RHEL/CentOS Firewall Configuration](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-using_firewalls)

## Notes

- This guide assumes you're using a RHEL-based distribution (RHEL, CentOS, Oracle Linux, etc.)
- For Debian/Ubuntu systems, replace `yum` with `apt` and `httpd` with `apache2`
- Always ensure your security groups/network ACLs also allow inbound traffic on port 80