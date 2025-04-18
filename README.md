# HashiCorp Vault Development Environment on AWS EC2

This guide provides step-by-step instructions for setting up a HashiCorp Vault development environment on an AWS EC2 instance using Ubuntu. This setup is intended for **development, testing, and integration purposes only** and utilizes Vault's insecure development mode.

**What is Vault?** HashiCorp Vault is a tool for securely accessing secrets. A secret is anything that you want to tightly control access to, such as API keys, passwords, or certificates.

**Goal:** To quickly stand up a functional Vault server accessible via UI and CLI, configure basic secrets, and set up AppRole authentication for programmatic access (e.g., by Terraform or Ansible).

## Prerequisites

*   An AWS Account.
*   An EC2 Key Pair for SSH access.
*   An SSH client installed on your local machine.
*   Basic familiarity with AWS EC2, Security Groups, and the Linux command line.

## 1. Create an AWS EC2 Instance

1.  **Launch Instance:** Go to the AWS EC2 console and launch a new instance.
    *   Choose an **Ubuntu** AMI (e.g., Ubuntu Server 22.04 LTS).
    *   Select an appropriate instance type (e.g., `t2.micro` or `t3.micro` is usually sufficient for development).
    *   Configure or select your existing EC2 Key Pair for SSH access.
2.  **Configure Security Group:** Create a new security group or modify an existing one associated with the instance. Ensure the following inbound rules are present:
    *   **Type:** SSH, **Protocol:** TCP, **Port Range:** 22, **Source:** Your IP address (Recommended for security) or `0.0.0.0/0` (Less secure).
    *   **Type:** Custom TCP, **Protocol:** TCP, **Port Range:** 8200, **Source:** Your IP address or `0.0.0.0/0` (Allows access to the Vault UI/API).
        *   **Warning:** Using `0.0.0.0/0` exposes the port to the entire internet. Restrict the source IP range as much as possible, especially if sensitive data might be involved even in development.
3.  **Launch** the instance and note its Public IPv4 address.

## 2. Install Vault on the EC2 Instance

1.  **Connect via SSH:**
    ```bash
    ssh -i /path/to/your-key.pem ubuntu@<your-ec2-public-ip>
    ```
    Replace `/path/to/your-key.pem` and `<your-ec2-public-ip>` accordingly.

2.  **Install Vault:** Execute the following commands to add the HashiCorp GPG key, add the official HashiCorp repository, update your package list, and install Vault.
    ```bash
    # Ensure package lists are up-to-date first
    sudo apt update

    # Add HashiCorp GPG key
    wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

    # Add HashiCorp repository
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

    # Update package list again and install Vault
    sudo apt update && sudo apt install vault
    ```

## 3. Start Vault in Development Mode

1.  **Start the Dev Server:**
    ```bash
    vault server -dev -dev-listen-address="0.0.0.0:8200"
    ```
    *   `-dev`: Starts Vault in an insecure development mode. Data is stored in-memory and lost when the process stops. It automatically unseals Vault.
    *   `-dev-listen-address="0.0.0.0:8200"`: Makes Vault listen on all network interfaces on port 8200, allowing access via the EC2 instance's public IP.

    **🚨 WARNING: Development mode is inherently insecure and MUST NOT be used for production environments or sensitive data.** It bypasses security features like TLS encryption (by default) and uses a single unseal key.

2.  **Note Credentials:** After starting, Vault will output crucial information. **Save these immediately**:
    ```
    ==> Vault server configuration:

                 Api Address: http://0.0.0.0:8200
                     Cgo: disabled
              Cluster Address: https://127.0.0.1:8201
                   Listener 1: tcp (addr: "0.0.0.0:8200", cluster address: "0.0.0.0:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
                  Log Level: info
                      Mlock: supported: true, enabled: false
              Recovery Mode: false
                    Storage: inmem (HA available)
                    Version: Vault vX.Y.Z // Your version may differ
                Version Sha: <commit_sha>

    ==> Vault server started! Log data will stream in below:

    ...[logs]...

    WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
    and starts unsealed with a single unseal key. The root token is already
    authenticated to the CLI, so you can immediately begin using Vault.

    You may need to set the following environment variable:
        export VAULT_ADDR='http://<your-ec2-public-ip>:8200' // Adjusted for public access

    The unseal key and root token are displayed below in case you want to
    seal/unseal the Vault or re-authenticate.

    Unseal Key: <YOUR_UNSEAL_KEY>
    Root Token: <YOUR_ROOT_TOKEN>

    Development mode should NOT be used in production environments.
    ```
    *   **Root Token:** This is like the superuser password. You'll need it for the initial UI login and administrative CLI tasks.
    *   **Unseal Key:** In dev mode, Vault starts unsealed, but this key is provided for reference. In a production setup, multiple keys are typically required to unseal Vault after restarts.

    **Keep the terminal running**, as closing it will stop the Vault dev server and erase all data. Consider using `tmux` or `screen` for long-running sessions.

## 4. Access Vault UI

1.  **Ensure Security Group Allows Access:** Double-check that your EC2 instance's Security Group allows inbound traffic on TCP port 8200 from your IP address (or `0.0.0.0/0`).
2.  **Navigate in Browser:** Open your web browser and go to:
    ```
    http://<your-ec2-public-ip>:8200
    ```
3.  **Log In:** On the login page, select the **Token** method and enter the **Root Token** you saved from the terminal output.

    ![log in](https://github.com/user-attachments/assets/f6e7b8a8-fd57-494a-89ce-dd1a75235d2e)

    You should now be logged into the Vault UI.

    ![UI](https://github.com/user-attachments/assets/7776fd6b-34b5-4c84-8656-56b48ec107c1)

## 5. Create a Secret in KV Store

Vault uses Secrets Engines to manage different types of secrets. The Key/Value (KV) engine is commonly used for static secrets.

1.  Navigate to **Secrets** in the left sidebar.
2.  Click **Enable new engine**.
    ![Secrets Engine](https://github.com/user-attachments/assets/1eeb64fc-7f9a-4ab6-8035-f51d906a99b8)
3.  Select **KV** (Key/Value) secrets engine.
    ![Enable a Secrets Engine](https://github.com/user-attachments/assets/3fbf0ce2-3ddc-4bf1-a85d-2b0de5d2c60f)
4.  Keep the default path (e.g., `kv/`) or specify a custom one. Choose **Version 2** (recommended for versioning and soft deletes) unless you have specific reasons for v1. Click **Enable Engine**.
    ![image](https://github.com/user-attachments/assets/bbaced97-3318-4807-b41e-4ffb6dce34f8)
5.  Click on the newly enabled engine (e.g., `kv/`).
    ![image](https://github.com/user-attachments/assets/199dd612-9abe-4725-bf76-77f3d1880148)
6.  Click **Create secret**.
    ![Create a Secret](https://github.com/user-attachments/assets/eb596c4e-16b7-4889-bb1b-c28eaa23ffc8)
7.  Enter a **Path for this secret** (e.g., `myapp/config`) and add your key-value data (e.g., `api_key = 12345abcd`). Click **Save**.
    ![Secret Engine](https://github.com/user-attachments/assets/b5286de9-7835-477c-b85b-ddc018db72fb)

You have now successfully stored a secret in Vault!

## 6. Configure AppRole Authentication for Automation

AppRole is an authentication method designed for applications and automated workflows (like Terraform or Ansible) to authenticate with Vault. It uses a **Role ID** (public, like a username) and a **Secret ID** (private, like a password).

Configuring AppRole authentication, including defining policies and roles, is typically done via the CLI or API for precision and automation.

1.  **Open a NEW Terminal/SSH Session:** Connect to your EC2 instance again in a separate terminal window. This allows you to run `vault` CLI commands while the server runs in the first terminal.

2.  **Set Vault Address and Token:** Configure the CLI to talk to your Vault server.
    ```bash
    # Set the address (use the public IP)
    export VAULT_ADDR='http://<your-ec2-public-ip>:8200'

    # Set the root token for administrative commands
    export VAULT_TOKEN='<YOUR_ROOT_TOKEN>'

    # Verify connection (optional)
    vault status
    ```
    Replace `<your-ec2-public-ip>` and `<YOUR_ROOT_TOKEN>` with your actual values.

3.  **Enable AppRole Auth Method:** (If not already enabled)
    ```bash
    vault auth enable approle
    ```
    *Output: `Success! Enabled approle auth method at: approle/`*

4.  **Create an Access Policy:** Define what actions an authenticated AppRole identity can perform. Create a file named `terraform-policy.hcl` (or use a heredoc as shown) with the desired permissions.

    ```bash
    # Example policy granting broad access to KV v2 secrets under 'kv/data/'
    # and the ability for the AppRole to create tokens for itself.
    # Refine these permissions based on the principle of least privilege for production.
    vault policy write terraform-policy - <<EOF
    # Allow listing and reading capabilities generally (adjust as needed)
    path "*" {
      capabilities = ["list"]
    }

    # Allow full CRUDL operations on secrets within the 'kv' mount path
    # Note: For KV v2, operations happen under the 'data/' subpath.
    path "kv/data/*" {
      capabilities = ["create", "read", "update", "delete", "list"]
    }
    # If you created secrets under a different path or engine, adjust accordingly.
    # Example for default 'secret/' KV v1 path:
    # path "secret/data/*" {
    #   capabilities = ["create", "read", "update", "delete", "list"]
    # }

    # Allow the AppRole to create tokens for itself (necessary for login)
    path "auth/token/create" {
      capabilities = ["create", "read", "update", "list"]
    }
    EOF
    ```
    *Output: `Success! Uploaded policy: terraform-policy`*

5.  **Create the AppRole Role:** Define an AppRole named `terraform-role` and attach the policy created above. Customize TTLs (Time-To-Live) and usage counts as needed.

    ```bash
    vault write auth/approle/role/terraform-role \
        secret_id_ttl=10m          # Secret ID is valid for 10 minutes
        token_num_uses=10          # Resulting token can be used 10 times
        token_ttl=20m              # Resulting token is valid for 20 minutes
        token_max_ttl=30m          # Max lifetime for the resulting token is 30 minutes
        secret_id_num_uses=40      # Secret ID can be used 40 times to log in
        token_policies="terraform-policy" # Attach the policy created earlier
    ```
    *Output: `Success! Data written to: auth/approle/role/terraform-role`*

    ![AppRole](https://github.com/user-attachments/assets/1d502d5d-57b9-44e5-93d2-acd5478d3e3c)
    ![AppRole Details](https://github.com/user-attachments/assets/b34a5a6a-3742-4897-a664-e1259ba9cdc8)
    ![Role Config](https://github.com/user-attachments/assets/ef3e54f4-b40b-4c98-a8ca-7c3551e51af3)

6.  **Retrieve AppRole Credentials:** Get the Role ID and generate a Secret ID.

    *   **Get Role ID:**
        ```bash
        vault read auth/approle/role/terraform-role/role-id
        ```
        *Sample Output:*
        ```
        Key        Value
        ---        -----
        role_id    <YOUR_ROLE_ID>
        ```
        **Save the `role_id` value.**

    *   **Generate Secret ID:**
        ```bash
        vault write -f auth/approle/role/terraform-role/secret-id
        ```
        *Sample Output:*
        ```
        Key                   Value
        ---                   -----
        secret_id             <YOUR_SECRET_ID>
        secret_id_accessor    <accessor_value>
        secret_id_ttl         10m
        ```
        **Save the `secret_id` value securely.** This value is sensitive and is not retrievable again via the API.

    **🚨 Important:** Treat the `Secret ID` like a password. Store it securely and limit its exposure. The `Role ID` is less sensitive but still required for authentication.

## 7. Using AppRole Credentials in Automation

Tools like Terraform or Ansible have providers/modules that support AppRole authentication. You would typically provide the `VAULT_ADDR`, the `Role ID`, and the `Secret ID` (often via environment variables or secure configuration) to allow these tools to authenticate with Vault and retrieve the secrets they need based on the associated policy (`terraform-policy` in this example).

## Conclusion

You have successfully set up a basic HashiCorp Vault development server on AWS EC2, accessed its UI, stored a secret, and configured AppRole authentication for programmatic access.

**Remember:**
*   This setup uses **development mode** and is **not suitable for production**.
*   The Vault server process must remain running (e.g., using `tmux` or `screen`) for it to be accessible. Stopping the process will erase all data in dev mode.
*   Always secure your Root Token and Secret IDs carefully.
*   Refine policies according to the principle of least privilege.

For production deployments, explore Vault's documentation on storage backends (like Consul or S3), high availability, TLS configuration, and operational best practices.
