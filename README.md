## Instructions for installing and running HashiCorp Vault on an AWS EC2 instance.

### 1. Create an AWS EC2 Instance with Ubuntu

### 2. Install Vault on the EC2 Instance

1. SSH into your EC2 instance:
   ```bash
   ssh -i your-key.pem ubuntu@your-ec2-public-ip
   ```

2. Install Vault by running the following commands:
   ```bash
   wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
   sudo apt update && sudo apt install vault
   ```

### 3. Start Vault

1. To start Vault in development mode, run:
   ```bash
   vault server -dev -dev-listen-address="0.0.0.0:8200"
   ```

   This starts Vault in development mode and listens on all IPs at port 8200.

   > **Warning**: Development mode should NOT be used in production installations.

2. After running the command, you will see output similar to the following, which includes the **Unseal Key** and **Root Token**:

   ```
   The unseal key and root token are displayed below in case you want to
   seal/unseal the Vault or re-authenticate.

   Unseal Key: Dvq9kuZoZ9Vj**********LBaCpaBUV+L6c1mxO8s5Y=
   Root Token: hvs.DcJ****P61qBZ2M****5rfs7
   ```

   - **Unseal Key**: Used to unseal Vault after it has been sealed.
   - **Root Token**: The token will be used to log into the Vault UI with root access.

### 4. Access Vault from Browser

1. Open the **Security Groups** section in AWS EC2 instance's settings.
2. Add an inbound rule to allow traffic on port 8200:
   - Type: **Custom TCP**
   - Port: **8200**
   - Source: **0.0.0.0/0** (or restrict to IP)
   
3. Access Vault from the browser:
   ```
   http://<ec2-public-ip>:8200
   ```

   Use the **Root Token** from the terminal output to log in as the root user.

![log in](https://github.com/user-attachments/assets/f6e7b8a8-fd57-494a-89ce-dd1a75235d2e)

![UI](https://github.com/user-attachments/assets/7776fd6b-34b5-4c84-8656-56b48ec107c1)

![Secrets Engine](https://github.com/user-attachments/assets/1eeb64fc-7f9a-4ab6-8035-f51d906a99b8)

![Enable a Secrets Engine](https://github.com/user-attachments/assets/3fbf0ce2-3ddc-4bf1-a85d-2b0de5d2c60f)

![image](https://github.com/user-attachments/assets/bbaced97-3318-4807-b41e-4ffb6dce34f8)

![image](https://github.com/user-attachments/assets/199dd612-9abe-4725-bf76-77f3d1880148)

Create a Secret in KV
![image](https://github.com/user-attachments/assets/eb596c4e-16b7-4889-bb1b-c28eaa23ffc8)

![image](https://github.com/user-attachments/assets/b5286de9-7835-477c-b85b-ddc018db72fb)

### How can we grant access for Terraform or Ansible, we need to create a roll inside the Hashicorp Vault 
(It is similar to the concept of IAM Roles and for the IAM Roles we grant the policies. Consider Access as an IAM Roles and Policies as policies)
![image](https://github.com/user-attachments/assets/a1afce53-1ba6-41a9-aa6b-53ae391c9896)

Use AppRole base auth for Ansible and Terraform
![image](https://github.com/user-attachments/assets/1d502d5d-57b9-44e5-93d2-acd5478d3e3c)

![image](https://github.com/user-attachments/assets/b34a5a6a-3742-4897-a664-e1259ba9cdc8)

![image](https://github.com/user-attachments/assets/ef3e54f4-b40b-4c98-a8ca-7c3551e51af3)

We can not create any roles through the user interface, Use CLI for it...

## Configure Terraform to Read Secrets from Vault

To configure Terraform to authenticate and read secrets from Vault, follow these steps to enable and configure AppRole authentication in Vault:

### 1. Enable AppRole Authentication

To enable the AppRole authentication method in Vault, use the Vault CLI:

```bash
vault auth enable approle
```

This command enables the AppRole authentication method in Vault.

### 2. Create an AppRole

First, create a policy that will define the capabilities for your AppRole. Use the following policy definition to allow access to necessary paths:

```bash
vault policy write terraform - <<EOF
path "*" {
  capabilities = ["list", "read"]
}

path "secrets/data/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "kv/data/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "secret/data/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "auth/token/create" {
  capabilities = ["create", "read", "update", "list"]
}
EOF
```

Then, create the AppRole with the desired authentication settings:

```bash
vault write auth/approle/role/terraform \
    secret_id_ttl=10m \
    token_num_uses=10 \
    token_ttl=20m \
    token_max_ttl=30m \
    secret_id_num_uses=40 \
    token_policies=terraform
```

### 3. Generate Role ID and Secret ID

After creating the AppRole, you need to generate a **Role ID** and **Secret ID**.

#### a. Generate Role ID

To generate the **Role ID**, run the following command:

```bash
vault read auth/approle/role/terraform/role-id
```

Save the **Role ID** for use in your Terraform configuration.

#### b. Generate Secret ID

To generate a **Secret ID**, run the following command:

```bash
vault write -f auth/approle/role/terraform/secret-id
```

This command will generate a **Secret ID** and provide it in the response. Save the **Secret ID** securely, as it will be used to authenticate Terraform.
