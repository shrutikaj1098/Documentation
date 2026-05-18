# Complete SSH Jumphost Connection Flow Documentation

CODEBASE: https://github.fkinternal.com/Flipkart/secengg-deployment/blob/master/ansibles/roles/jumphost/tasks/jumphost-config.yml

## Overview
This document explains the complete step-by-step process of connecting to a target VM (`10.51.146.128`) through a jumphost (`global.jumphost-prod.fkcloud.in`) using SSH certificates and ProxyJump configuration.

## Architecture Diagram
```
┌─────────────────┐    ┌─────────────────────────────┐    ┌─────────────────┐
│   Your Laptop   │───▶│      Jumphost Server       │───▶│   Target VM     │
│                 │    │ global.jumphost-prod.fkcloud │    │  10.51.146.128  │
│ ┌─────────────┐ │    │                             │    │                 │
│ │ ~/.ssh/     │ │    │ ┌─────────────────────────┐ │    │ ┌─────────────┐ │
│ │ config      │ │    │ │ /etc/ssh/trusted-user-  │ │    │ │ ~/.ssh/      │ │
│ │ jumphost/   │ │    │ │ ca-keys.pem              │ │    │ │ authorized_  │ │
│ │ default     │ │    │ └─────────────────────────┘ │    │ │ keys         │ │
│ └─────────────┘ │    │ ┌─────────────────────────┐ │    │ └─────────────┘ │
│ ┌─────────────┐ │    │ │ /etc/sshd/sshd-banner   │ │    │                 │
│ │ id_rsa      │ │    │ └─────────────────────────┘ │    │                 │
│ │ id_rsa-cert │ │    │ ┌─────────────────────────┐ │    │                 │
│ └─────────────┘ │    │ │ ssh_host_rsa_key-cert.pub│ │    │                 │
└─────────────────┘    │ └─────────────────────────┘ │    └─────────────────┘
                       └─────────────────────────────┘
```

## Phase 1: Local SSH Configuration Setup

### 1.1 Configuration File Loading
```
debug1: Reading configuration data /Users/shrutika.a/.ssh/config
debug1: Reading configuration data /Users/shrutika.a/.ssh/jumphost/default
debug1: /Users/shrutika.a/.ssh/jumphost/default line 2: Applying options for 10.*
```

**What Happens:**
- SSH reads your main config file `~/.ssh/config`
- SSH reads jumphost-specific config `~/.ssh/jumphost/default`
- Line 2 of jumphost config contains: `Host 10.*` pattern match
- This line applies ProxyJump settings for all IPs starting with `10.`

**Local Config Content (Not in codebase):**
```bash
# ~/.ssh/jumphost/default
Host 10.*
    ProxyJump global.jumphost-prod.fkcloud.in
    User shrutika.a
```

### 1.2 ProxyJump to ProxyCommand Conversion
```
debug1: Setting implicit ProxyCommand from ProxyJump: ssh -v -W '[%h]:%p' global.jumphost-prod.fkcloud.in
```

**What Happens:**
- SSH automatically converts `ProxyJump` to `ProxyCommand`
- `%h` gets replaced with target hostname (`10.51.146.128`)
- `%p` gets replaced with target port (`22`)
- Result: `ssh -v -W '[10.51.146.128]:22' global.jumphost-prod.fkcloud.in`

### 1.3 Identity File Discovery
```
debug1: identity file /Users/shrutika.a/.ssh/id_rsa type 0
debug1: identity file /Users/shrutika.a/.ssh/id_rsa-cert type 4
debug1: identity file /Users/shrutika.a/.ssh/id_ed25519 type 3
```

**What Happens:**
- SSH scans `~/.ssh/` directory for available keys
- Finds RSA private key (`type 0`) and RSA certificate (`type 4`)
- Finds Ed25519 private key (`type 3`) but no certificate
- `type -1` means file not found for other key types

## Phase 2: Jumphost Connection Establishment

### 2.1 Proxy Command Execution
```
debug1: Executing proxy command: exec ssh -v -W '[10.51.146.128]:22' global.jumphost-prod.fkcloud.in
```

**What Happens:**
- SSH executes the proxy command to connect to jumphost
- This creates a tunnel: Your Laptop → Jumphost → Target VM
- The `-W` flag enables standard input/output forwarding

### 2.2 Jumphost Connection Setup
```
debug1: Connecting to global.jumphost-prod.fkcloud.in port 22.
debug1: Connection established.
debug1: Remote protocol version 2.0, remote software version OpenSSH_9.2p1 Debian-2+deb12u10
```

**What Happens:**
- TCP connection established to jumphost on port 22
- SSH protocol negotiation begins
- Jumphost identifies itself as OpenSSH_9.2p1 on Debian

### 2.3 Key Exchange Algorithm Selection
```
debug1: kex: algorithm: sntrup761x25519-sha512
debug1: kex: host key algorithm: rsa-sha2-512-cert-v01@openssh.com
```

**What Happens:**
- Client and server negotiate encryption algorithms
- `sntrup761x25519-sha512` - Post-quantum resistant key exchange
- `rsa-sha2-512-cert-v01@openssh.com` - Certificate-based host authentication

### 2.4 Jumphost Host Certificate Validation
```
debug1: Server host certificate: ssh-rsa-cert-v01@openssh.com SHA256:PDct3tD4BRKmeweMDTC5/X6KWL7qKXVAZuZMGrMxUTI, serial 13241258266001262133 ID "vault-token-3c372dded0f80512a67b078c0d30b9fd7e8a58beea29754066e64c1ab3315132" CA ssh-rsa SHA256:hiK0wbA+YOFNzvhORal5VzIUPf5j8Gm9u4D4La5FHhc valid from 2026-03-25T15:43:54 to 2126-03-01T15:44:24
```

**Codebase Connection:**
This certificate was created by the Ansible code in `jumphost-config.yml`:

```yaml
# Lines 6-23: Host Key Signing Process
- name: Read Public Key of Host
  shell: cat /etc/ssh/ssh_host_rsa_key.pub
  register: host_pub_key

- name: Download Signed Host Key
  uri:
    url: "http://{{vips.vault}}/v1/{{vaultPath}}/sign/{{hostrole}}"
    method: POST
    body: {"public_key":"{{host_pub_key.stdout}}", "cert_type":"host", "ttl": "3153600000"}
```

**What Happens:**
1. **Ansible** read the jumphost's public key: `ssh_host_rsa_key.pub`
2. **Ansible** sent it to Vault for signing
3. **Vault** created a certificate with:
   - **Serial**: 13241258266001262133
   - **Valid from**: 2026-03-25T15:43:54
   - **Valid until**: 2126-03-01T15:44:24 (100 years!)
   - **CA fingerprint**: SHA256:hiK0wbA+YOFNzvhORal5VzIUPf5j8Gm9u4D4La5FHhc

### 2.5 Certificate Trust Validation
```
debug1: Host 'global.jumphost-prod.fkcloud.in' is known and matches the RSA-CERT host certificate.
debug1: Found CA key in /Users/shrutika.a/.ssh/known_hosts:1
```

**What Happens:**
- Your SSH client validates the certificate signature
- Checks against CA public key stored in your `known_hosts`
- Confirms the certificate is legitimately signed

### 2.6 Banner Display
```
*__________________________________________*
* You`re passing through Jumphost Tunnels. *
*------------------------------------------*
```

**Codebase Connection:**
This banner was created by Ansible in `jumphost-config.yml`:

```yaml
# Lines 25-32: Banner Creation
- lineinfile:
    path: /etc/sshd/sshd-banner
    line: '{{ item }}'
    create: yes
  with_items:
    - '*__________________________________________*'
    - '* You`re passing through Jumphost Tunnels. *'
    - '*------------------------------------------*'
```

## Phase 3: User Authentication to Jumphost

### 3.1 Authentication Method Selection
```
debug1: Authentications that can continue: publickey
debug1: Next authentication method: publickey
```

**What Happens:**
- Jumphost offers publickey authentication only
- Password authentication is disabled (security best practice)

### 3.2 SSH Agent Key Discovery
```
debug1: get_agent_identities: bound agent to hostkey
debug1: get_agent_identities: agent returned 3 keys
debug1: Will attempt key: /Users/shrutika.a/.ssh/id_ed25519 ED25519 SHA256:ZEFbQBxjxTgyU89pbSWJnhTjz8HcgmbNhLZjHvRjTmM agent
debug1: Will attempt key: teleport:teleport.dbcreds-playground.fkcloud.in:teleport-pg:shrutika.a RSA-CERT SHA256:TlO/3Md/yctzzJd6ByaPNnyThUZNiARtKqBZVQRYn5k agent
debug1: Will attempt key: /Users/shrutika.a/.ssh/id_rsa RSA-CERT SHA256:I1YIjFzba3lU7IOgnKTt7uUrRBnG/7WqPPjwdqFkUUM
```

**What Happens:**
- SSH client queries SSH agent for available keys
- Agent returns 3 keys: Ed25519, Teleport certificate, RSA certificate
- Client will try these keys in order

### 3.3 Certificate Authentication Attempts
```
debug1: Offering public key: /Users/shrutika.a/.ssh/id_ed25519 ED25519 SHA256:ZEFbQBxjxTgyU89pbSWJnhTjz8HcgmbNhLZjHvRjTmM agent
debug1: Authentications that can continue: publickey
debug1: Offering public key: /Users/shrutika.a/.ssh/id_rsa RSA-CERT SHA256:I1YIjFzba3lU7IOgnKTt7uUrRBnG/7WqPPjwdqFkUUM
debug1: Server accepts key: /Users/shrutika.a/.ssh/id_rsa RSA-CERT SHA256:I1YIjFzba3lU7IOgnKTt7uUrRBnG/7WqPPjwdqFkUUM
```

**What Happens:**
- Client tries Ed25519 key first, but jumphost doesn't accept it
- Client tries RSA certificate, which jumphost accepts
- Certificate validation happens on jumphost side

### 3.4 Jumphost Certificate Validation
**Codebase Connection:**
The jumphost validates your certificate against the CA public key downloaded by Ansible:

```yaml
# Lines 1-4: CA Public Key Setup
- name: Download Public Key of CA
  get_url:
    url: "http://{{vips.vault}}/v1/{{vaultPath}}/public_key"
    dest: /etc/ssh/trusted-user-ca-keys.pem
```

**What Happens on Jumphost:**
1. Jumphost receives your RSA certificate
2. Checks if it's signed by trusted CA (from `/etc/ssh/trusted-user-ca-keys.pem`)
3. Validates certificate signature and expiration
4. Extracts user identity (`shrutika.a`) from certificate
5. Grants access

### 3.5 Successful Jumphost Authentication
```
Authenticated to global.jumphost-prod.fkcloud.in ([10.121.82.6]:22) using "publickey".
```

**What Happens:**
- You are now authenticated to the jumphost
- SSH tunnel is established through jumphost
- Ready to connect to target VM

## Phase 4: Target VM Connection Through Tunnel

### 4.1 Tunnel Forwarding Setup
```
debug1: channel_connect_stdio_fwd: 10.51.146.128:22
debug1: channel 0: new stdio-forward [stdio-forward] (inactive timeout: 0)
```

**What Happens:**
- SSH creates forwarding channel through the tunnel
- Forwards your SSH connection to `10.51.146.128:22`
- All communication now flows: Laptop → Jumphost → Target VM

### 4.2 Target VM Protocol Negotiation
```
debug1: Remote protocol version 2.0, remote software version OpenSSH_8.4p1 Debian-5+deb11u1
debug1: kex: algorithm: curve25519-sha256
debug1: kex: host key algorithm: ssh-ed25519
```

**What Happens:**
- New SSH protocol negotiation with target VM
- Target VM uses older OpenSSH_8.4p1
- Different algorithms selected (curve25519-sha256, ssh-ed25519)

### 4.3 Target VM Host Key Validation
```
debug1: Server host key: ssh-ed25519 SHA256:2rg9Ym/TgQ1HO9XCuyTQibPsjeyqLEC71H/F7ANQa0k
debug1: Host '10.51.146.128' is known and matches the ED25519 host key.
debug1: Found key in /Users/shrutika.a/.ssh/known_hosts:6
```

**What Happens:**
- Target VM presents its host key (not certificate-based)
- Your client validates against known_hosts file
- Key matches previous connection, so it's trusted

### 4.4 Target VM User Authentication
```
debug1: Authentications that can continue: publickey
debug1: Offering public key: /Users/shrutika.a/.ssh/id_rsa RSA SHA256:I1YIjFzba3lU7IOgnKTt7uUrRBnG/7WqPPjwdqFkUUM
debug1: Server accepts key: /Users/shrutika.a/.ssh/id_rsa RSA SHA256:I1YIjFzba3lU7IOgnKTt7uUrRBnG/7WqPPjwdqFkUUM
```

**What Happens:**
- Target VM accepts your RSA key (not certificate)
- Uses traditional `authorized_keys` mechanism
- Your public key is stored in `/home/shrutika.a/.ssh/authorized_keys`

### 4.5 Successful Target VM Authentication
```
Authenticated to 10.51.146.128 (via proxy) using "publickey".
```

**What Happens:**
- You are now authenticated to the target VM
- Connection flows through jumphost tunnel
- Ready for interactive session

## Phase 5: Interactive Session Establishment

### 5.1 Session Setup
```
debug1: channel 0: new session [client-session] (inactive timeout: 0)
debug1: Entering interactive session.
debug1: pledge: filesystem
```

**What Happens:**
- SSH creates interactive session channel
- Sets up security constraints (pledge)
- Prepares for shell access

### 5.2 Environment and Permissions
```
debug1: Remote: /home/shrutika.a/.ssh/authorized_keys:1: key options: agent-forwarding port-forwarding pty user-rc x11-forwarding
debug1: Sending environment.
debug1: channel 0: setting env LC_CTYPE = "UTF-8"
```

**What Happens:**
- Target VM reads authorized_keys options
- Grants permissions for agent forwarding, port forwarding, etc.
- Sets up environment variables

### 5.3 Successful Login
```
Linux secengg-service-metrics-8555296 5.10.0-23-cloud-amd64 #1 SMP Debian 5.10.179-1 (2023-05-12) x86_64

Last login: Mon May 11 13:08:52 2026 from 10.53.114.16
shrutika.a@secengg-service-metrics-8555296:~$
```

**What Happens:**
- You now have shell access to the target VM
- Connection is fully established through jumphost
- Ready to execute commands

## Codebase Integration Summary

### Ansible Role: `jumphost-config.yml`
```yaml
# Lines 1-4: CA Trust Setup
- name: Download Public Key of CA
  dest: /etc/ssh/trusted-user-ca-keys.pem

# Lines 6-23: Host Certificate Signing  
- name: Read Public Key of Host
- name: Download Signed Host Key

# Lines 25-32: Banner Creation
- lineinfile:
    path: /etc/sshd/sshd-banner

# Lines 34-35: SSH Service Restart
- name: Restart SSH
  service: name=ssh state=restarted
```

### Ansible Role: `access-logs.yml`
```yaml
# Lines 17-18: PAM Integration
- name: Setup sshd login script
  lineinfile: dest=/etc/pam.d/sshd line="session optional pam_exec.so /usr/share/jumphost/sshd_login.sh"

# Lines 4-6: Login Script Deployment
- {src: sshd_login.sh, dst: /usr/share/jumphost/sshd_login.sh}
```

### Login Script: `sshd_login.sh`
```bash
#!/bin/bash
ppid=`ps -o ppid= $$ | xargs`
case "$PAM_TYPE" in
    open_session)
        echo "$(date '+%b %d %H:%M:%S') $HOSTNAME sshd[$ppid]: User $PAM_USER logged in from $PAM_RHOST" >> /var/log/flipkart/jumphost/access.log
        ;;
esac
```

## Security Benefits

1. **Certificate-Based Authentication**: No need to manage individual authorized_keys
2. **Centralized CA Control**: Vault manages all certificate signing
3. **Automatic Expiration**: Certificates have built-in expiration dates
4. **Audit Logging**: All connections logged via PAM script
5. **Tunnel Security**: All traffic flows through monitored jumphost

## Troubleshooting Guide

### Common Issues:
1. **Certificate Expired**: Check certificate validity dates
2. **CA Trust Missing**: Verify `/etc/ssh/trusted-user-ca-keys.pem` on jumphost
3. **ProxyJump Config**: Check local SSH config for correct ProxyJump settings
4. **Host Key Mismatch**: Verify target VM host key in known_hosts

### Debug Commands:
```bash
# Check jumphost CA trust
ssh -v global.jumphost-prod.fkcloud.in

# Check certificate details  
ssh-keygen -L -f ~/.ssh/id_rsa-cert.pub

# Verify jumphost configuration
ansible jumphost -m command -a "cat /etc/ssh/trusted-user-ca-keys.pem"
```

---

*Document created on: May 18, 2026*
*Author: shrutika.a*
*Version: 1.0*
