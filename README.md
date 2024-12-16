# HashiCorp Vault Installation on AWS EC2

Instructions for installing and running HashiCorp Vault on an AWS EC2 instance.

## Steps

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
