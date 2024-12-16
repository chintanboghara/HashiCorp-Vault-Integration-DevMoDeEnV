# Vault Integration Development Environments

## Instructions for Installing and Running HashiCorp Vault on an AWS EC2 Instance

### 1. Create an AWS EC2 Instance with Ubuntu
- Launch an EC2 instance with an Ubuntu AMI.
- Make sure to configure the appropriate security group to allow SSH access and inbound traffic on port 8200 for Vault access.

### 2. Install Vault on the EC2 Instance

1. **SSH into your EC2 instance**:
   ```bash
   ssh -i your-key.pem ubuntu@your-ec2-public-ip
   ```

2. **Install Vault by running the following commands**:
   ```bash
   wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
   sudo apt update && sudo apt install vault
   ```

### 3. Start Vault

1. **Start Vault in development mode**:
   ```bash
   vault server -dev -dev-listen-address="0.0.0.0:8200"
   ```

   This starts Vault in development mode and listens on all IPs at port 8200.

   **Warning**: Development mode should **NOT** be used in production environments.

2. After running the command, Vault will output the **Unseal Key** and **Root Token**. These are necessary for unsealing and accessing the Vault UI:
   ```
   The unseal key and root token are displayed below in case you want to
   seal/unseal the Vault or re-authenticate.

   Unseal Key: Dvq9kuZoZ9Vj**********LBaCpaBUV+L6c1mxO8s5Y=
   Root Token: hvs.DcJ****P61qBZ2M****5rfs7
   ```

   - **Unseal Key**: Used to unseal Vault after it has been sealed.
   - **Root Token**: Used for initial login to Vault with root access.

### 4. Access Vault from Browser

1. **Open the EC2 instance's Security Groups** and add an inbound rule to allow traffic on port 8200:
   - Type: **Custom TCP**
   - Port: **8200**
   - Source: **0.0.0.0/0** (or restrict to specific IPs)

2. **Access Vault** by opening your browser and navigating to:
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

### 5. Create a Secret in KV

1. Navigate to the **Secrets Engines** section in the Vault UI.
2. Enable a **KV Secrets Engine** and create a secret.

   ![Create a Secret](https://github.com/user-attachments/assets/eb596c4e-16b7-4889-bb1b-c28eaa23ffc8)

   ![Secret Engine](https://github.com/user-attachments/assets/b5286de9-7835-477c-b85b-ddc018db72fb)

### 6. Grant Access to Terraform or Ansible via Vault

Similar to **IAM Roles** in AWS, in Vault, we create **roles** and assign **policies** to manage access. This is how we control access for Terraform and Ansible:

   ![image](https://github.com/user-attachments/assets/a1afce53-1ba6-41a9-aa6b-53ae391c9896)

   Use **AppRole-based authentication** for Terraform and Ansible integration:
   ![AppRole](https://github.com/user-attachments/assets/1d502d5d-57b9-44e5-93d2-acd5478d3e3c)

   ![AppRole Details](https://github.com/user-attachments/assets/b34a5a6a-3742-4897-a664-e1259ba9cdc8)

   ![Role Config](https://github.com/user-attachments/assets/ef3e54f4-b40b-4c98-a8ca-7c3551e51af3)

### 7. Create Roles Using the CLI

We cannot create roles via the Vault UI. Use the Vault CLI for this:

1. **Enable AppRole Authentication**:
   ```bash
   vault auth enable approle
   ```

2. **Create a Policy**:
   Create a policy that allows the AppRole to access necessary paths:
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

3. **Create the AppRole**:
   ```bash
   vault write auth/approle/role/terraform \
       secret_id_ttl=10m \
       token_num_uses=10 \
       token_ttl=20m \
       token_max_ttl=30m \
       secret_id_num_uses=40 \
       token_policies=terraform
   ```

4. **Generate Role ID and Secret ID**:

   - **Generate Role ID**:
     ```bash
     vault read auth/approle/role/terraform/role-id
     ```

   - **Generate Secret ID**:
     ```bash
     vault write -f auth/approle/role/terraform/secret-id
     ```

   Save both **Role ID** and **Secret ID** securely. These will be used for authentication in Terraform.
