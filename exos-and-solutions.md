# Automation with Python — Exercises & Solutions

---

## Exercise 1: Working with Subnets in AWS

**Task:**
- Get all the subnets in your default region
- Print the subnet IDs

**Solution:**

```python
import boto3

ec2 = boto3.client('ec2')
subnets = ec2.describe_subnets()
for subnet in subnets["Subnets"]:
    if subnet["DefaultForAz"]:
        print(subnet["SubnetId"])
```

---

## Exercise 2: Working with IAM in AWS

**Task:**
- Get all the IAM users in your AWS account
- For each user, print out the name of the user and when they were last active (hint: Password Last Used attribute)
- Print out the user ID and name of the user who was active the most recently

**Solution:**

```python
import boto3

iam = boto3.client('iam')
iam_users = iam.list_users()

last_active_user = iam_users["Users"][0]

for iam_user in iam_users["Users"]:
    print(iam_user["UserName"])
    print(iam_user["PasswordLastUsed"])
    print("---------------------------")
    
    if last_active_user["PasswordLastUsed"] < iam_user["PasswordLastUsed"]:
        last_active_user = iam_user

print("Last active user:")
print(last_active_user["UserId"])
print(last_active_user["UserName"])
print(last_active_user["PasswordLastUsed"])
```

---

## Exercise 3: Automate Running and Monitoring Application on EC2 instance

**Task:**
Write a Python program which automatically creates an EC2 instance, installs Docker inside and starts an Nginx application as Docker container and starts monitoring the application as a scheduled task. Write the program with the following steps:
- Start EC2 instance in default VPC
- Wait until the EC2 server is fully initialized
- Install Docker on the EC2 server
- Start nginx container
- Open port for nginx to be accessible from browser
- Create a scheduled function that sends a request to the nginx application and checks the status is OK
- If status is not OK 5 times in a row, it restarts the nginx application

**Solution:**

**Prerequisites:**
Do the following manually to prepare your AWS region for the script execution:
- Open the SSH port 22 in the default security group in your default VPC
- Create a key-pair for your ec2 instance. Download the private key of the key-pair and set its access permission to 400 mode
- Set the values for: image_id, key_name, instance_type and ssh_private_key_path in your python script

```python
from distutils import command
import boto3
import time
import paramiko
import requests
import schedule

ec2_resource = boto3.resource('ec2')
ec2_client = boto3.client('ec2')

# set all needed variable values
image_id = 'ami-031eb8d942193d84f'
key_name = 'boto3-server-key'
instance_type = 't2.small'

# the pem file must have restricted 400 permissions: chmod 400 absolute-path/boto3-server-key.pem
ssh_private_key_path = '/Users/nana/Downloads/boto3-server-key.pem' 
ssh_user = 'ec2-user'
ssh_host = '' # will be set dynamically below

# Start EC2 instance in default VPC

# check if we have already created this instance using instance name
response = ec2_client.describe_instances(
    Filters=[
        {
            'Name': 'tag:Name',
            'Values': [
                'my-server',
            ]
        },
    ]
) 

instance_already_exists = len(response["Reservations"]) != 0 and len(response["Reservations"][0]["Instances"]) != 0
instance_id = ""

if not instance_already_exists: 
    print("Creating a new ec2 instance")
    ec2_creation_result = ec2_resource.create_instances(
        ImageId=image_id, 
        KeyName=key_name, 
        MinCount=1, 
        MaxCount=1, 
        InstanceType=instance_type,
        TagSpecifications=[
            {
                'ResourceType': 'instance',
                'Tags': [
                    {
                        'Key': 'Name',
                        'Value': 'my-server'
                    },
                ]
            },
        ],
    )
    instance = ec2_creation_result[0]
    instance_id = instance.id
else:
    instance = response["Reservations"][0]["Instances"][0]
    instance_id = instance["InstanceId"]
    print("Instance already exists")

# Wait until the EC2 server is fully initialized
ec2_instance_fully_initialised = False

while not ec2_instance_fully_initialised:
    print("Getting instance status")
    statuses = ec2_client.describe_instance_status(
        InstanceIds = [instance_id]
    )
    if len(statuses['InstanceStatuses']) != 0:
        ec2_status = statuses['InstanceStatuses'][0]

        ins_status = ec2_status['InstanceStatus']['Status']
        sys_status = ec2_status['SystemStatus']['Status']
        state = ec2_status['InstanceState']['Name']
        ec2_instance_fully_initialised = ins_status == 'ok' and sys_status == 'ok' and state == 'running'
    if not ec2_instance_fully_initialised:
        print("waiting for 30 seconds")
        time.sleep(30)

print("Instance fully initialised")

# get the instance's public ip address
response = ec2_client.describe_instances(
    Filters=[
        {
            'Name': 'tag:Name',
            'Values': [
                'my-server',
            ]
        },
    ]
) 
instance = response["Reservations"][0]["Instances"][0]
ssh_host = instance["PublicIpAddress"]

# Install Docker on the EC2 server & start nginx container
commands_to_execute = [
    'sudo yum update -y && sudo yum install -y docker',
    'sudo systemctl start docker',
    'sudo usermod -aG docker ec2-user',
    'docker run -d -p 8080:80 --name nginx nginx'
]

# connect to EC2 server
print("Connecting to the server")
print(f"public ip: {ssh_host}")
ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect(hostname=ssh_host, username=ssh_user, key_filename=ssh_private_key_path)

# install docker & start nginx 
for command in commands_to_execute:
    stdin, stdout, stderr = ssh.exec_command(command)
    print(stdout.readlines())

ssh.close()

# Open port 8080 on nginx server, if not already open
sg_list = ec2_client.describe_security_groups(
    GroupNames=['default']
)

port_open = False
for permission in sg_list['SecurityGroups'][0]['IpPermissions']:
    print(permission)
    # some permissions don't have FromPort set
    if 'FromPort' in permission and permission['FromPort'] == 8080:
        port_open = True

if not port_open:
    sg_response = ec2_client.authorize_security_group_ingress(
        FromPort=8080,
        ToPort=8080,
        GroupName='default',
        CidrIp='0.0.0.0/0',
        IpProtocol='tcp'
    )

# Scheduled function to check nginx application status and reload if not OK 5x in a row
app_not_accessible_count = 0

def restart_container():
    print('Restarting the application...')
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh.connect(hostname=ssh_host, username=ssh_user, key_filename=ssh_private_key_path)
    stdin, stdout, stderr = ssh.exec_command('docker start nginx')
    print(stdout.readlines())
    ssh.close()
    # reset the count
    global app_not_accessible_count
    app_not_accessible_count = 0
    print(app_not_accessible_count)

def monitor_application():
    global app_not_accessible_count
    try:
        response = requests.get(f"http://{ssh_host}:8080")
        if response.status_code == 200:
            print('Application is running successfully!')
        else:
            print('Application Down. Fix it!')
            app_not_accessible_count += 1
            if app_not_accessible_count == 5:
                restart_container()
    except Exception as ex:
        print(f'Connection error happened: {ex}')
        print('Application not accessible at all')
        app_not_accessible_count += 1
        if app_not_accessible_count == 5:
            restart_container()
        return "test"
    
schedule.every(10).seconds.do(monitor_application)  

while True:
    schedule.run_pending()
```

---

## Exercise 4: Working with ECR in AWS

**Task:**
- Get all the repositories in ECR
- Print the name of each repository
- Choose one specific repository, and for that repository list all the image tags inside, sorted by date, with the most recent image tag on top

**Solution:**

```python
import boto3
from operator import itemgetter

ecr_client = boto3.client('ecr')

# Get all ECR repos and print names
repos = ecr_client.describe_repositories()['repositories']
for repo in repos:
    print(repo['repositoryName'])

print("-----------------------")

# For one specific repo, get all the images and print them out sorted by date

# replace with your own repo-name
repo_name = "java-app"
images = ecr_client.describe_images(
    repositoryName=repo_name
)

image_tags = []

for image in images['imageDetails']:
    image_tags.append({
        'tag': image['imageTags'],
        'pushed_at': image['imagePushedAt']
    })

images_sorted = sorted(image_tags, key=itemgetter("pushed_at"), reverse=True)
for image in images_sorted:
    print(image)
```

---

## Exercise 5: Python in Jenkins Pipeline

**Task:**
Create a Jenkins job that fetches all the available images from your application's ECR repository using Python. It allows the user to select the image from the list through user input and deploys the selected image to the EC2 server using Python.

**Instructions:**
Do the following preparation manually:
- Start EC2 instance and install Docker on it
- Install Python, Pip and all needed Python dependencies in Jenkins
- Create 3 Docker images with tags 1.0, 2.0, 3.0 from one of the previous projects

Once all the above is configured, create a Jenkins Pipeline with the following steps:
1. Fetch all 3 images from the ECR repository (using Python)
2. Let the user select the image from the list
3. SSH into the EC2 server (using Python)
4. Run docker login to authenticate with ECR repository (using Python)
5. Start the container from the selected image from step 2 on EC2 instance (using Python)
6. Validate that the application was successfully started and is accessible by sending a request to the application (using Python)

**Solution:**

**Manual Setup Tasks:**
```bash
# Install Python inside Jenkins server
apt-get install python3
apt-get install pip
pip install boto3
pip install paramiko
pip install requests

# Create credentials in Jenkins 
"jenkins_aws_access_key_id" - Secret Text
"jenkins_aws_secret_access_key" - Secret Text
"ssh-creds" - SSH Username with private key
"ecr-repo-pwd" - Secret Text

# NOTE: you will have to approve usage of "split" function in script. 
# You will see the link to approval inside the build console logs
```

**Jenkins Pipeline:**
In the exercise repository you will find the Jenkinsfile that executes 3 Python scripts for different stages:
- `get-images.py`
- `deploy.py`
- `validate.py`

**Environment Variables Configuration:**
Before executing the Jenkins pipeline, set the following environment variable values inside Jenkinsfile:
- `ECR_REPO_NAME`
- `EC2_SERVER`
- `ECR_REGISTRY`
- `CONTAINER_PORT`
- `HOST_PORT`
- `AWS_DEFAULT_REGION`

**Pipeline Steps:**
1. **Fetch Images Stage**: Uses `get-images.py` to connect to ECR and retrieve available image tags
2. **User Input Stage**: Presents the list of images to user for selection via Jenkins input parameter
3. **Deploy Stage**: Uses `deploy.py` to SSH into EC2, authenticate with ECR, and deploy selected container
4. **Validate Stage**: Uses `validate.py` to send HTTP requests to verify application is running correctly

This creates a complete CI/CD pipeline that allows for interactive deployment of different application versions from ECR to EC2 instances.
