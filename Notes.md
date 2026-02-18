Post install:
Create new user and assign builtin admin
Set home and sudo
Create API Key and assign to new user - Set this in AWS
Commands:
# 1. Download the installer
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# 2. Unzip it (you may need to run 'sudo apt install unzip' first if you don't have it)
unzip awscliv2.zip

# 3. Run the installer
sudo ./aws/install

aws configure --profile ansible-ssm-reader
sudo apt install python3-boto3 python3-botocore
pip install boto3 botocore
ansible-galaxy collection install amazon.aws
