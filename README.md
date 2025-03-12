# my-open-webui
Install your own instance of Open WebUI for personal use

## Assumptions and pre-requisites
- You are using Windows machine
- You have access to an AWS account

## Design principles
- Keep cost lowest
- KISS - Keep it simple, stupid 

## What you will get?
- Open WebUI
- Portainer (Web based Docker management)
- Code-server (VSCode on EC2 in your browser)
- LiteLLM (Gateway to Bedrock)
- Caddy (reverse proxy and authetication server)

## SETUP GUIDE
### Install Terraform
- Download installer from (Use AMD64):  
https://developer.hashicorp.com/terraform/install
- Install terraform

### Install Git for Windows
- Download and install from:  
https://git-scm.com/downloads/win

### Install AWS CLI for Windows. Follow:
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

### Clone this project
- Open cmd window and execute:
```bat
git clone https://github.com/kashodiya/my-open-webui.git
cd my-open-webui
```

### Login to AWS
- Either set the AWS credentials env vars
- OR, setup profile and set AWS_DEFAULT_PROFILE
- Ensure env var AWS_REGION is set

### Set your VPC ID for terraform
- Find out your VPC Id using this command 
```bat
aws ec2 describe-vpcs  
```
- Create new file: terraform\terraform.tfvars.json
- Enter VPC Id and an unused subnet range for a new subnet
```json
{
    "vpc_id": "vpc-your-vpc-id-here",
    "allowed_source_ips": [
        "3.83.200.219/32"
    ]
}
```

### Ensure you have Internet Gateway associated with VPC
- Check if you already have Internet Gateway using this command:  
```bat
aws ec2 describe-internet-gateways
```
- If you do not have Internet Gatewy, create one using:  
```bat
aws ec2 create-internet-gateway  
```
- Note down internet gateway id and associate it to the VPC using this command:  
```bat
aws ec2 attach-internet-gateway --internet-gateway-id igw-xxxxxxxx --vpc-id vpc-xxxxxxxx
```

### Find our your IP address
- Go to:  
https://whatismyipaddress.com/
- Copy IPv4

### Update values in terraform\terraform.tfvars.json file
- Update allowed_source_ips array by replacing your IP address in there.
- Optional:
    - If you also want to access Open WebUI from some other network/laptop make sure that you add that machine's public IP address to the array.
    - Update these items if needed:
        - ami (this must be Amazon Linux os)
        - instance_type
        - project_id

### Set LiteLLM API Key
- Decide a key (short random string/numbers)
- Edit docker\docker-compose.yml and update following 2 values.  
LITELLM_API_KEY  
OPENAI_API_KEY  
- Ensure that both the values are same

### Init and apply terraform
```bat
cd terraform  
terraform init  
terraform apply  
```
- Check the plan and when ask for Enter a value, enter yes, hit Enter key

### Create launcher
- Create a bat file (launch.bat or whatever you like) on your desktop with this content:  
```bat
@echo off  
set AWS_DEFAULT_PROFILE=your-aws-profile  
start cmd /k "cd /d D:\Users\full-path-to-project-code && call scripts\start-dev.bat"  
```
- Whenever you want to start working on this project, just double click this bat file!  
- Read the info presented in the cmd window!  
- It offers following shortcuts:  
```ini
tfa = Terraform apply  
sshe = SSH into EC2  
ec2 = Start EC2  
ec2x = Stop EC2  
open-webui = Opens Open WebUI in Browser  
portainer = Opens Portainer in Browser  
code-server = Opens code-server in Browser  
litellm = Opens LiteLLM in Browser  
rkh = Remove known SSH host  
```

### SSH into EC2 (hard way)
- Find Elastic IP address from terraform\set-tf-output-2-env-var.bat file.
- SSH into ec2 using shortcut from launcher OR,
- SSH into the EC2 server using this command:
```bat
set PROJECT_DIR=path/to/your/project/folder  
set ELASTIC_IP=your.elastic.ip.address  
ssh -i %PROJECT_DIR%\keys\private_key.pem ec2-user@%ELASTIC_IP%
```

### SSH into EC2 (easy way)
- Run scripts\start-dev.bat
- Use this shortcut (read - ssh to ec2):  
sshe  

### Verify the install
- To see complete cloud init log:  
sudo tail -f /var/log/cloud-init-output.log  
- To see only user data script log:  
sudo tail -f /var/log/user-data.log  

### Set admin user password for Open WebUI
- Run this command in cmd window:  
start http://%ELASTIC_IP%:8101
- Sign up for admin user

### Request access to bedrock models
- Open docker\open-webui\litellm-config.yml and request model access for each models mentioned in the config.
- Login to AWS Console
- Request models access by going to:  
https://us-east-1.console.aws.amazon.com/bedrock/home?region=us-east-1#/modelaccess


### Use Open WebUI
- Open open-webui in browser using shortcut
open-webui  
- OR, Open code-server in browser using url:  
https://your.public.ip.address:7101

### Use code server (VSCode to EC2 server in your Browser!)
- Get code-server password by SSH into the server and running:  
cat /home/ec2-user/.config/code-server/config.yaml
- Copy password from that
- Open code-server in browser using shortcut
code-server  
- OR, Open code-server in browser using url:  
https://your.public.ip.address:7104


### Setup Open WebUI connections 
TODO

## Maintenance and operations

### Stop EC2 server
aws ec2 stop-instances --instance-ids %INSTANCE_ID%

### Start EC2 server
aws ec2 start-instances --instance-ids %INSTANCE_ID%

### Delete everything on AWS that was created via this project
cd terraform  
terraform destroy  

### How to upgrade Open WebUI to new version?
TODO: Add more details 
- SSH into EC2
- Stop Docker containers using command:  
```bash
cd open-webui  
docker-compose down
```
- Delete Open WebUI image
- Start docker compose
```bash
docker-compose up -d
```

### How to add more Bedrock models?
- Make sure that you have requested access to the model
- SSH into EC2 server
- Edit docker/litellm-config.yml
    - Add a model in the model list
- Restart LiteLLM container  
```bash
cd docker  
docker-compose restart litellm  
```

### How to manage Open WebUI users?
TODO

### How to check server logs?
- SSH into EC2 server  
```bash
cd docker
docker logs -f open-webui
docker logs -f litellm
```

### How to recreate EC2?
- WARNING: This will delete your EC2 and all data inside it!
- Execute following commands  
```bat
cd terraform  
terraform taint aws_instance.main_instance  
```
- After you create new EC2, before doing SSH into EC2, do this:
```bat
ssh-keygen -R %ELASTIC_IP%  
```

### Commands related to the code-server
```bash
sudo systemctl status code-server@$USER
sudo systemctl stop code-server@$USER
sudo cat /usr/lib/systemd/system/code-server@.service
sudo vi /usr/lib/systemd/system/code-server@.service
sudo systemctl daemon-reload
sudo systemctl restart code-server@$USER
```

- Read password
cat /home/ec2-user/.config/code-server/config.yaml


## Tips and tricks
### Tip for setting your development environment
- Prefer to use AWS profile instead of directly using AWS credentials in enviroment variables
- Create a bat file on your desktop with this content:  
@echo off  
set AWS_DEFAULT_PROFILE=your-aws-profile  
start cmd /k "cd /d D:\Users\full-path-to-project-code && call scripts\start-dev.bat"  
- Whenever you want to start working on this project, just double click this bat file!  
- Read the info presented in the cmd window!  
- It offers following shortcuts:  
tfa = Terraform apply  
sshe = SSH into EC2  
ec2 = Start EC2  
ec2x = Stop EC2  
open-webui = Opens Open WebUI in Browser  
portainer = Opens Portainer in Browser  
code-server = Opens code-server in Browser  
litellm = Opens LiteLLM in Browser  
rkh = Remove known SSH host  


## Resources and references
### Open WebUI
- [Docs](https://docs.openwebui.com/)
- [Home page](https://openwebui.com/)

### LiteLLM
- [Docs](https://docs.litellm.ai/docs/)
- [Home page](https://www.litellm.ai/)

### Code server
- [GitHub page](https://github.com/coder/code-server)

### Portainer
- [GitHub page](https://github.com/portainer/portainer)
- [Home page](https://www.portainer.io/)

### Caddy
- [Home page](https://caddyserver.com/docs/quick-starts/reverse-proxy)

## Troubleshooting
### When doing terraform apply: Error: No matching Internet Gateway found
- You should create Internet gatewat and attach to your VPC (see the instructions above)



## TODO:
How to reset password of Open WebUI
Figure out VPC, should not be checked in

