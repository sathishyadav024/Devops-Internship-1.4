
# `Deploy Medusa to AWS-EC2 Using GitHub Actions`

This Project shows how to deploy a Medusa backend to an Amazon EC2 instance using GitHub Actions. The workflow is triggered every time there is a push to the main branch.



## 🔗 Links

[![linkedin](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/sathish-gurka)


## Authors

- [@GurkaSathish](https://github.com/sathishyadav024)


## Pre-Requisites

- `GitHub`
- `AWS account`
- `An EC2 instance running Ubuntu with a public IP.`
- `SSH access to the EC2 instance.`
- `GitHub repository with the necessary secrets configured.`

## GitHub Actions Workflow


Below is the workflow configuration (.github/workflows/main.yml). This workflow performs the following steps:

- Checks out the repository.
- Sets up SSH access to the EC2 instance using the provided SSH key.
- SSHs into the EC2 instance and installs required dependencies.
- Installs and Configures PostgreSQL,redis-server and Medusa-CLI.
- Creates new Medusa project
- Runs migrations and starts the Medusa backend. 

```
name: Deploy Medusa to EC2

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v3

    - name: Set up SSH for EC2
      env:
        EC2_SSH_KEY: ${{ secrets.EC2_SSH_KEY }}
      run: |
        echo "$EC2_SSH_KEY" | base64 -d > /tmp/id_rsa
        chmod 600 /tmp/id_rsa

    - name: Deploy to EC2
      env:
        EC2_IP: ${{ secrets.EC2_IP }}
        EC2_USER: ${{ secrets.EC2_USER }}
      run: |
        ssh -o StrictHostKeyChecking=no -i /tmp/id_rsa $EC2_USER@$EC2_IP << 'EOF'
          # Update system and install necessary dependencies
          sudo apt update && sudo apt upgrade -y
          sudo apt install -y git
          sudo apt install -y nodejs npm git postgresql redis-server
          sudo systemctl enable postgresql
          sudo systemctl start postgresql
          sudo systemctl enable redis-server

          # Set up PostgreSQL for Medusa-backend
          sudo -u postgres psql -c "CREATE USER medusa_user WITH PASSWORD 'sathish';" || true
          sudo -u postgres psql -c "CREATE DATABASE medusa_db;" || true
          sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE medusa_db TO medusa_user;" || true

           # Grant additional permissions to the user
          sudo -u postgres psql -d medusa_db -c "GRANT ALL PRIVILEGES ON SCHEMA public TO medusa_user;"

          # Configure PostgreSQL authentication method
          sudo sh -c 'echo "host all all 0.0.0.0/0 md5" >> /etc/postgresql/16/main/pg_hba.conf'

           # Reload PostgreSQL to apply configuration changes
          sudo systemctl reload postgresql


          # Install Medusa CLI
          sudo npm install -g @medusajs/medusa-cli

          # Create a new Medusa project
          mkdir medusa-backend && cd medusa-backend
          sudo medusa new . --skip-db

          # Update the .env file to connect to the correct PostgreSQL database
          echo "DATABASE_URL=postgres://medusa_user:sathish@localhost:5432/medusa_db" | sudo tee .env
          sudo chown $USER:$USER .env && chmod 644 .env
          
          # run migrations
          sudo medusa migrations run
          # set admin user-name and password 
          sudo medusa user --email sathishyadavdiploma@gmail.com --password sathish

          # Start Medusa backend
          nohup sudo medusa start &> medusa.log &
        EOF
```
##  Run the Workflow

Push code to the main branch of your repository:

```
git add .
git commit -m "Deploy Medusa to EC2"
git push origin main

```
## Check Logs

The output of the Medusa backend is saved to medusa.log on the EC2 instance. To view the logs:
```
ssh -i /path/to/id_rsa ubuntu@<EC2_IP>
cat /path/to/medusa-backend/medusa.log

```
## Access Medusa Backend

Once the deployment completes, you should be able to access the Medusa backend on port 9000 of the EC2 instance:
```
http://<EC2_IP>:9000/app/login

```
## `Result`

`Successfully installed and deployed the Medusa-Backend using GithubActions on AWS-EC2` 
