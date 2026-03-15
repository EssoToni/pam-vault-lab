# PAM & Secure Access Lab: WireGuard VPN + HashiCorp Vault


## Overview

This project demonstrates a **Privileged Access Management (PAM)** solution using **HashiCorp Vault** combined with **WireGuard VPN**. The goal is to show how a client can securely access a protected web service without ever storing credentials locally. Vault acts as the central PAM component, dynamically providing application credentials only after the client authenticates via the VPN. The design requires **no modification to the existing firewall (nftables)** and proves that the server never discloses secrets, while the client learns credentials only at access time.

## Objectives

- Demonstrate a **zero‑trust** approach: the server never stores or divulges secrets to the client.
- Prove that the client does **not know credentials in advance**.
- Show Vault operating as a **central PAM** for application access.
- Enable access to a protected service **exclusively via secrets delivered by Vault**.
- Avoid any changes to the **nftables** firewall configuration.

## Technologies Used

| Component          | Purpose                                                       |
|--------------------|---------------------------------------------------------------|
| **WireGuard**      | Secure VPN tunnel between client and server                   |
| **HashiCorp Vault**| Central secret management and PAM                              |
| **Apache2**        | Web server hosting a protected resource (HTTP Basic Auth)     |
| **htpasswd**       | Create and manage HTTP Basic authentication credentials       |
| **curl** & **jq**  | API interaction with Vault and testing                        |
| **nftables**       | Existing firewall (unchanged)                                  |

## Architecture

[VPN Client] ─── WireGuard Tunnel ─── [Security VM]
│
┌────────────┴────────────┐
│ │
┌─────▼─────┐ ┌───────▼──────┐
│ Vault │ │ Apache │
│ (PAM) │ │ (protected │
└───────────┘ │ resource) │
└──────────────┘
text


- The client connects to the Security VM via **WireGuard**.
- Vault runs on the same VM and stores the application credentials (username/password) as a secret.
- Apache serves a protected endpoint (`/pam`) that requires **HTTP Basic Authentication**.
- The client retrieves credentials from Vault **only after VPN authentication**, then uses them to access the protected page.

## Setup & Installation

### 1. Server-Side: Prepare the Protected Web Service (on Security VM)

Install Apache and create a password‑protected area:

```bash
sudo apt install -y apache2 apache2-utils
sudo systemctl enable --now apache2

# Create an application user
sudo htpasswd -c /etc/apache2/.htpasswd pamuser
# Password: PamWeb2026
```

Configure a virtual host to protect the /pam location:
```bash

sudo tee /etc/apache2/sites-available/pam.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName pam.lab
    DocumentRoot /var/www/html

    <Location "/pam">
        AuthType Basic
        AuthName "PAM Protected Area"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
    </Location>
</VirtualHost>
EOF

sudo a2ensite pam
sudo systemctl reload apache2
```

Create the protected resource:
```bash

sudo mkdir -p /var/www/html/pam
sudo tee /var/www/html/pam/index.html > /dev/null <<EOF
<h1>PAM Access Granted</h1>
<p>Authentication successful via Vault</p>
EOF
```

    Note: Apache already listens on the server; no nftables changes are required.

### 2. Server-Side: Store the Secret in Vault

Set Vault address and token, then store the credentials:
```bash

export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=labroot

curl -H "X-Vault-Token: $VAULT_TOKEN" \
     -X POST \
     -d '{"data":{"web_user":"pamuser","web_pass":"PamWeb2026"}}' \
     $VAULT_ADDR/v1/secret/data/webapp | jq .
```

### 3. Client-Side: Access Without Local Secrets (VPN connected)

Verify network connectivity:
```bash

curl http://10.10.10.1
# → Apache default page (public)
```

Attempt direct access to the protected area (should fail):
```bash

curl http://10.10.10.1/pam
# → 401 Unauthorized
```

Retrieve credentials from Vault:
```bash

export VAULT_ADDR=http://10.10.10.1:8200
export VAULT_TOKEN=labroot

USER=$(curl -s -H "X-Vault-Token: $VAULT_TOKEN" \
           $VAULT_ADDR/v1/secret/data/webapp | jq -r '.data.data.web_user')
PASS=$(curl -s -H "X-Vault-Token: $VAULT_TOKEN" \
           $VAULT_ADDR/v1/secret/data/webapp | jq -r '.data.data.web_pass')
```

Now access the protected page using the dynamically obtained credentials:
```bash

curl -u $USER:$PASS http://10.10.10.1/pam
```

✅ Expected output:
```text

<h1>PAM Access Granted</h1>
<p>Authentication successful via Vault</p>
```

## Key Demonstration Points
Element	| Result
|--|--|
Secret stored persistently on client |	❌
Hardcoded credentials anywhere |	❌
Access outside VPN	| ❌
Access without Vault |	❌
Access via Vault |	✅
Firewall (nftables) modification |	❌

Vault effectively acts as the PAM provider.
### Secret Rotation (Proof of PAM)

    1. Change the application password on the server:
    ```bash

    sudo htpasswd /etc/apache2/.htpasswd pamuser
    # New password: PamWeb2026v2
    ```

    2. Update the secret in Vault:
    ```bash

    curl -H "X-Vault-Token: $VAULT_TOKEN" \
         -X POST \
         -d '{"data":{"web_user":"pamuser","web_pass":"PamWeb2026v2"}}' \
         $VAULT_ADDR/v1/secret/data/webapp
    ```

    3. Attempt access with the old credentials – fails.

    4. Retrieve new credentials from Vault – access succeeds.

✅ Rotation occurs without any modification to the client configuration.
## Results

  Successfully demonstrated a Vault‑centric PAM architecture.

  Client obtains credentials dynamically and only when needed.

  No secrets are stored on the client machine.

  Existing firewall rules remain untouched.

  Secret rotation is seamless and does not require client intervention.

## Lessons Learned

  Vault as a PAM centralises secret management and enforces dynamic access.

  WireGuard provides a lightweight, secure tunnel that complements Vault’s authentication.

  Separation of concerns: the VPN authenticates the client; Vault authorises access to applications.

  No firewall changes simplifies deployment in environments with strict network policies.

  Secret rotation is simple and can be automated, proving the PAM model’s resilience.

## Future Improvements

  Integrate Vault’s dynamic database credentials instead of static secrets.

  Add Vault policies to restrict which clients can read specific secrets.

  Replace static htpasswd with a Vault‑aware authentication module (e.g., using Vault Agent).

  Deploy the service in a containerised environment (Docker) for easier reproducibility.

  Implement automatic secret rotation with scheduled Vault updates.

  Monitor access logs to detect anomalous behaviour.

---
Author: Esso Maléki TONINZIBA
