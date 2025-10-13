# OCI Vault Secret Retrieval Script

This Python script retrieves secrets from Oracle Cloud Infrastructure (OCI) Vault using Instance Principals authentication. It's designed to run on OCI compute instances without requiring hard-coded credentials.

## Overview

The script uses OCI Instance Principals to securely authenticate and retrieve secrets from OCI Vault. This approach eliminates the need to store API keys or credentials in your application code.

## Copyright Notice

```
COPYRIGHT (c) 2022 ORACLE
THIS SAMPLE CODE IS PROVIDED FOR EDUCATIONAL PURPOSES OR
TO ASSIST YOUR DEVELOPMENT OR ADMINISTRATION EFFORTS AND
PROVIDED "AS IS" AND IS NOT SUPPORTED BY ORACLE CORPORATION.
License: http://www.apache.org/licenses/LICENSE-2.0.html
```

## Prerequisites

- OCI Compute Instance with Instance Principal enabled
- Access to an OCI Vault with a configured secret
- Python 3.9 or higher
- Oracle Linux (OL9 recommended)

## Installation

### Step 1: Install Python and OCI CLI

Connect to your compute instance and run the following commands:

```bash
sudo dnf -y install oraclelinux-developer-release-el9
sudo dnf install python39-oci-cli
```

### Step 2: Download the Script

Download the Python script from the repository:

```bash
wget https://raw.githubusercontent.com/ou-developers/oci-vaultoperations/main/getsecret.py
```

## Configuration

### Step 3: Update the Secret OCID

1. Open the script in a text editor:

```bash
vi getsecret.py
```

2. Locate the `secret_id` variable near the top of the file:

```python
# Replace secret_id value below with the ocid of your secret
secret_id = "ocid1.vaultsecret.oc1.<my_secret_ocid>"
```

3. Replace the placeholder with your actual secret OCID:

```python
secret_id = "ocid1.vaultsecret.oc1.iad.xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

**Note:** To find your secret OCID:
- Navigate to your Vault in the OCI Console
- Click on the Secret link from the resources section
- Filter by your assigned compartment
- Click on your secret key and copy the OCID

4. Save and exit the editor:
   - Press `Esc` then type `:w` and press `Enter` to save
   - Press `Esc` then type `:q` and press `Enter` to exit

### Step 4: Make the Script Executable

```bash
chmod u+x getsecret.py
```

## Usage

Run the script to retrieve your secret:

```bash
python getsecret.py
```

The script will output the secret content stored in your OCI Vault.

## How It Works

1. **Authentication**: The script uses Instance Principals to authenticate without requiring API keys
2. **Connection**: Establishes a connection to OCI Secrets service
3. **Retrieval**: Fetches the secret bundle from the specified secret OCID
4. **Decoding**: Decodes the base64-encoded secret content
5. **Output**: Prints the secret value to stdout

## Key Functions

- `read_secret_value(secret_client, secret_id)`: Retrieves and decodes the secret from OCI Vault

## Security Benefits

Using Instance Principals and OCI Vault provides several advantages:

- No hard-coded credentials in application code
- Centralized secret management
- Automatic credential rotation support
- Fine-grained access control through IAM policies
- Encryption at rest and in transit

## Troubleshooting

If you encounter authentication errors:
- Verify Instance Principal is enabled on your compute instance
- Check that the instance has appropriate IAM policies to access the Vault
- Ensure the secret OCID is correct and accessible

## License

Apache License 2.0 - http://www.apache.org/licenses/LICENSE-2.0.html

## Additional Resources

For more information about OCI Vault and Instance Principals, refer to the [Oracle Cloud Infrastructure Documentation](https://docs.oracle.com/en-us/iaas/Content/KeyManagement/home.htm).
