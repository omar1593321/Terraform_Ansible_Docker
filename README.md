# Automating AWS Infrastructure with Terraform & Ansible: MongoDB and Mongo Express Deployment

🚀 Project Overview

  This project demonstrates how to use Terraform for AWS infrastructure provisioning and Ansible for automated configuration management, deploying a fully functional MongoDB   and Mongo Express environment using Docker. 
  The goal is to automate everything—from creating the cloud infrastructure to setting up Docker and running containers—while ensuring a repeatable, scalable process for       future projects.

🛠️ Technologies Used

    Terraform: Infrastructure as Code (IaC) tool used to define and provision AWS resources.
    Ansible: Used for post-provisioning tasks like installing Docker and configuring the application environment.
    Docker: Containerization platform used to run MongoDB and Mongo Express services.
    AWS: Cloud platform where resources such as EC2 instances, VPCs, and security groups are created.

🌐 Infrastructure Design

  VPC:

    CIDR Block: 10.0.0.0/16
    Subnets: One subnet with a CIDR block of 10.0.10.0/24 located in the us-east-1a availability zone.
    Internet Gateway: Attached to allow outbound internet access for instances.

EC2 Instance:

    Instance Type: t2.micro for lightweight operations.
    AMI: Latest Amazon Linux 2 AMI for compatibility with Docker.
    Key Pair: SSH access enabled with the specified key pair.
    Security Groups: Inbound rules for SSH (port 22) and Mongo Express (port 8081).

Networking:

    Route Table: Contains routes for internal traffic within the VPC and an internet route via the Internet Gateway.
    Security Groups: Ensures only necessary traffic (SSH and Mongo Express) is allowed, minimizing the attack surface.
    
📝 Terraform Configuration

VPC, Subnet, and Internet Gateway Setup:

    A VPC is provisioned with a /16 CIDR block for network segmentation.
    A single subnet with /24 CIDR is created for deploying the EC2 instance.
    An Internet Gateway is attached to allow outbound traffic, essential for Docker image downloads.
   
EC2 Instance with Provisioner for Ansible:

    aws_instance creates an Amazon Linux 2 EC2 instance.
    The local-exec provisioner runs an Ansible playbook to automate software installation and container orchestration.

Security Group:

    Ingress rules allow SSH access and Mongo Express traffic on ports 22 and 8081 respectively. Egress rules allow outbound traffic.
    
🔧 Ansible Playbook: Automated Setup

  1. SSH Connection Waiter
```
- name: wait for ssh connection
  hosts: all
  tasks:
    - name: Wait 300 seconds for port 22 to become open and contain "OpenSSH"
      wait_for:
        port: 22
        host: '{{ (ansible_ssh_host|default(ansible_host))|default(inventory_hostname) }}'
        search_regex: OpenSSH
        delay: 10
```
What This Does:

    Goal: Before Ansible can configure the instance, it needs to ensure SSH access is available. This task waits for SSH (on port 22) to become available before proceeding.
    wait_for module: It checks whether the SSH service (specifically "OpenSSH") is running on the target EC2 instance. If the instance takes time to fully boot up, this module will delay the execution for up to 300 seconds, with a 10-second delay between checks.

Why It's Important:

    Ensures that Ansible doesn't attempt to run tasks before the EC2 instance is ready to accept connections.
    Prevents playbook failure by handling potential delays in instance startup.
  
  ـــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــ
  
2. Install Docker, Docker-Compose, and Docker-py
   ```
   - name: install docker, Docker-Compose, Docker-py
    hosts: all
    become: yes
    tasks:
    - name: Install Docker
      yum:
        name: docker
        update_cache: yes
        state: present
        
    - name: Install Docker-Compose
      get_url:
        url: https://github.com/docker/compose/releases/download/1.29.2/docker-compose-Linux-x86_64
        dest: /usr/local/bin/docker-compose
        mode: +x
        
    - name: Install Docker-py, Docker-Compose-py, ssl for docker login
      pip:
        name:
          - docker
          - Docker-Compose
          - urllib3==1.26.15

   ```
  What This Does:

    Install Docker: The yum module is used to install Docker on the EC2 instance (as you are likely using Amazon Linux or CentOS).
        update_cache: Ensures the package manager cache is updated before installing Docker.
        state: present: Ensures Docker is installed, but doesn't reinstall if it's already present.

    Install Docker-Compose:
        Since Docker-Compose is not available via yum, the get_url module downloads the binary directly from GitHub.
        mode: +x: Adds execution permissions to the docker-compose binary, allowing it to be run.

    Install Python Docker Libraries:
        The pip module installs necessary Python libraries:
            docker library: Used for interacting with Docker via Python.
            Docker-Compose library: Used for interacting with Docker Compose.
            urllib3==1.26.15: A required SSL library for secure communication (important for Docker login to private/public registries).

Why It's Important:

    Docker and Docker Compose are critical for container orchestration. By automating their installation, you ensure consistency across environments and avoid manual setup errors.
    Installing the Python libraries (docker and Docker-Compose) makes it possible to programmatically manage Docker containers using Ansible.

    
  ـــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــ
      
3. Start Docker Service
```
- name: Start Docker-Compose
  hosts: all
  become: yes
  tasks:
    - name: Start Docker-Compose
      systemd:
        name: docker
        state: started

```
What This Does:

    Start Docker Service:
        The systemd module is used to start the Docker service. Without this, Docker would be installed but not running, making it impossible to launch containers.

Why It's Important:

    Ensures that Docker is running so that you can manage containers. This step is essential because Docker doesn’t start automatically after installation.

  
  ـــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــ
    
4. Add EC2-User to Docker Group
```
- name: Add EC2-user to docker group
  hosts: all
  become: yes
  tasks:
    - name: Add EC2-user to docker group
      user:
        name: ec2-user
        groups: docker
        append: yes
    - name: Reconnect to the server
      meta: reset_connection

```
What This Does:

    Add User to Docker Group:
        The user module adds the ec2-user to the docker group, allowing it to run Docker commands without requiring sudo.
        append: yes: Ensures that the user is only added to the Docker group without losing membership in any other groups.
    Reconnect to Server:
        The meta: reset_connection is used to refresh the Ansible connection. When the user is added to the Docker group, the connection may need to be reset for the new group membership to take effect.

Why It's Important:

    Without adding ec2-user to the docker group, you would have to run Docker commands with sudo, which is not ideal for automation.
    Resetting the connection ensures that future Docker commands are run without requiring superuser privileges.
  
  ـــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــ
  
  5. Deploy Docker Containers with Docker-Compose
   ```
- name: Start Docker Container
      hosts: all
      vars_files:
         - vars
       tasks:
         - name: Copy docker_compose.yaml file
           copy:
             src: /home/omar/ansible_terra_project/docker-compose.yaml
             dest: /home/ec2-user/docker-compose.yaml
        
    - name: Docker Login
      community.docker.docker_login:
        registry_url: https://index.docker.io/v1/
        username: "your username"
        password: "{{docker_password}}"
        
    - name: Start Container from compose
      community.docker.docker_compose:
        project_src: /home/ec2-user/
   ```
What This Does:

    Copy Docker-Compose File:
        The copy module copies your docker-compose.yaml file from the Ansible control node to the EC2 instance.
        This file defines the services (MongoDB and Mongo Express) that Docker will manage.

    Docker Login:
        The community.docker.docker_login module logs into Docker Hub, allowing access to private/public images stored there.
        It uses your Docker Hub credentials (username, password) to authenticate against the registry.

    Start Containers with Docker Compose:
        The community.docker.docker_compose module runs docker-compose to start the MongoDB and Mongo Express services defined in your docker-compose.yaml file.
        project_src specifies the directory where the docker-compose.yaml file resides on the target EC2 instance.

Why It's Important:

    Automation: This ensures the Docker containers are deployed automatically on the EC2 instance without any manual intervention. MongoDB and Mongo Express services will start based on the configurations in the docker-compose.yaml.
    Login: Docker Hub authentication is essential if you're using private images or need access to specific versions of public images.   
    
Summary of Ansible Automation

    Hands-Off Approach: Once your Terraform provisions the infrastructure, Ansible takes over, ensuring the required software (Docker) is installed and configured.
    Consistency: Every time you run this playbook, the same steps are applied, ensuring a predictable setup.
    Efficiency: You don’t need to manually SSH into your EC2 instance to install Docker, start services, or manage containers—Ansible handles all of it.

By automating with Ansible, you streamline the entire deployment process, reducing manual work and ensuring consistency in your cloud environment.    

ـــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــــ

🐳 Docker Compose Services
MongoDB Service:

    MongoDB is configured with an admin username and password. The database data is stored in a Docker volume for persistence.

Mongo Express:

    Provides a web interface to manage MongoDB. This service is exposed on port 8081.
    
📈 Why Use This Setup?

  Scalable and Repeatable: With IaC (Terraform) and configuration management (Ansible), you can easily scale and redeploy this architecture in any AWS region.
  Automated End-to-End: From infrastructure to application setup, the entire process is automated, reducing human error and speeding up deployment time.
  Containerized Deployment: Docker ensures that the services are portable, consistent, and can run on any environment that supports containers.  

A GIF for the final build:

![ansible-tera](https://github.com/user-attachments/assets/813d78d7-393d-4032-be15-c8eeea5dbaaa)

Screen shot:


![Screenshot 2024-09-30 022629](https://github.com/user-attachments/assets/5b46b024-971e-4cd5-bc95-bc1797bc61f7)
