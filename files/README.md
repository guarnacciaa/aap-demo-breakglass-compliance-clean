# Files Directory

This directory contains sensitive files used by the AAP demo. **These files should NOT be committed to version control.**

## Required Files

### `breakglass_id_rsa`
The SSH **private key** for the break-glass account. This is used as a Machine Credential in AAP.

### `breakglass_id_rsa.pub`
The SSH **public key** for the break-glass account. This will be deployed to target hosts' `~/.ssh/authorized_keys`.

## How to Generate the SSH Key Pair

```bash
# Generate a new Ed25519 key pair (recommended)
ssh-keygen -t ed25519 -f breakglass_id_ed25519 -C "breakglass@emergency"

# Or generate an RSA key pair (legacy compatibility)
ssh-keygen -t rsa -b 4096 -f breakglass_id_rsa -C "breakglass@emergency"
```

## Security Recommendations

1. **Use Ansible Vault** to encrypt sensitive files:
   ```bash
   ansible-vault encrypt breakglass_id_rsa
   ```

2. **Use AAP Credential Store** to securely store credentials instead of files.

3. **Never commit** these files to Git. They are listed in `.gitignore`.

4. **Rotate keys regularly** as per your organization's security policy.

## Example Structure

```
files/
├── README.md           ← This file
├── breakglass_id_rsa   ← Private key (DO NOT COMMIT)
└── breakglass_id_rsa.pub ← Public key (optional to commit)
```



