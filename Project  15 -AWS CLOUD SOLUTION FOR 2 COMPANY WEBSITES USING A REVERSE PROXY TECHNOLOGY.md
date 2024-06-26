# AWS CLOUD SOLUTION FOR 2 COMPANY WEBSITES USING A REVERSE PROXY TECHNOLOGY


In this project we built a secure infrastructure inside AWS VPC (Virtual Private Cloud) network for a company that uses WordPress CMS for its main business website, and a Tooling Website for their DevOps team. As part of the company’s desire for improved security and performance, a decision was made to use a reverse proxy technology from NGINX to achieve this.


The infrastructure for both websites; WordPress and Tooling, is resilient to Web Server’s failures, can accomodate to increased traffic and, at the same time, has reasonable cost. Below is a pictoral representation of the Infrastructure.


![image 1](https://user-images.githubusercontent.com/85270361/179696615-46db7282-b380-4e0e-b603-a9c605045b52.PNG)



## STEP 1: SET UP ORGANISATION UNIT (OU) and  a SUB-ACCOUNT
1. Properly configure your AWS account and Organization Unit.

- Create an AWS Master account.

- Within the Root account, create a sub-account and name it DevOps. (You will need another email address to complete this)

- Within the Root account, create an AWS Organization Unit (OU). Name it Dev. (We will launch Dev resources in there)

- Move the DevOps account into the Dev OU.

![image 2](https://user-images.githubusercontent.com/85270361/179697261-87aac0af-e647-4d6c-82a8-ae8031995df9.PNG)


- Login to the newly created AWS account using the new email address.

2. Create a domain name for the company using Route 53 on AWS.


## STEP 2: CREATE AN AWS VPC


### Set Up a Virtual Private Network (VPC)

1. Create a VPC
![image 3](https://user-images.githubusercontent.com/85270361/179697707-c86d18bc-03ae-4d81-bbb5-a8cd334c589f.PNG)

2. Create subnets as shown in the architecture
![image 4](https://user-images.githubusercontent.com/85270361/179698044-778e8a45-0f38-45e4-9622-172cca779586.PNG)

3. Create a route table and associate it with public subnets
![image 5](https://user-images.githubusercontent.com/85270361/179698364-69152f2a-a06a-4ecf-83a1-9a595c28daad.PNG)

4. Create a route table and associate it with private subnets
![image 6](https://user-images.githubusercontent.com/85270361/179699126-97a1cc25-1853-4c0a-829d-c618ade5bb7d.PNG)

5. Create an Internet Gateway
![image 7](https://user-images.githubusercontent.com/85270361/179699426-a45e06be-9c54-4557-b1aa-ff54f91a7cdb.PNG)

6. Edit a route in public route table, and associate it with the Internet Gateway. (This is what allows a public subnet to be accisble from the Internet).
7. Created 3 Elastic IPs.
8. Created a Nat Gateway and assigned one of the Elastic IPs (*The other 2 was used by Bastion hosts).
9. Created security groups for different services in the architecture


- **Nginx Servers:** Access to Nginx should only be allowed from a Application Load balancer (ALB). At this point, we have not created a load balancer, therefore we will update the rules later. For now, just create it and put some dummy records as a place holder.

- **Bastion Servers:** Access to the Bastion servers should be allowed only from workstations that need to SSH into the bastion servers. Hence, you can use your workstation public IP address. (This is the IP address that you will use to SSH into the bastion servers). **On inbound rules, pick TCP (22) and choose "my IP" (to restrict access to only your device).**

- **Application Load Balancer:** ALB will be available from the Internet

- **Webservers:** Access to Webservers should only be allowed from the Nginx servers. Since we do not have the servers created yet, just put some dummy records as a place holder, we will update it later.

- **Data Layer:** Access to the Data layer, which is comprised of Amazon Relational Database Service (RDS) and Amazon Elastic File System (EFS) must be carefully desinged – only webservers should be able to connect to RDS, while Nginx and Webservers will have access to EFS Mountpoint.

### Set Up Compute Resources for Nginx

#### Provision EC2 Instances for Nginx

1. Create an EC2 Instance based on CentOS Amazon Machine Image (AMI) in any 2 Availability Zones (AZ) in any AWS Region close to the target users. Use EC2 instance of T2 family (e.g. t2.micro or similar)

2. Ensure that it has the following software installed: python, ntp,net-tools,vim,wget,telnet,epel-release,htop.

3. Create an AMI out of this instance

#### Prepare Launch Template For Nginx (One Per Subnet)

1. Make use of the AMI to set up a launch template

2. Ensure the Instances are launched into a public subnet (public subnet 1 or 2)

3. Assign appropriate security group (NGINX SG)

4. Add a userdata to install Nginx, clone ou project config repo and carry out some configuration tasks.
```
#!/bin/bash
yum install -y nginx
systemctl start nginx
systemctl enable nginx
git clone https://github.com/proteusxlr/ACS-project-config.git
mv /ACS-project-config/reverse.conf /etc/nginx/
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf-distro
cd /etc/nginx/
touch nginx.conf
sed -n 'w nginx.conf' reverse.conf
systemctl restart nginx
rm -rf reverse.conf
rm -rf /ACS-project-config
```

#### Configure Target Groups
Go to Target Groups section and create a new target group.
1. Select Instances as the target type
2. Ensure the protocol HTTPS on secure TLS port 443
3. Ensure that the health check path is /healthstatus
4. Register Nginx Instances as targets
5. Ensure that health check passes for the target group
![image 8](https://user-images.githubusercontent.com/85270361/179699959-91bc7b60-664b-4cbc-9ac3-62333a3524ec.PNG)


![image 9](https://user-images.githubusercontent.com/85270361/179700228-d4de926a-9bab-4209-afa8-2503e2cbe82e.PNG)


#### Configure Autoscaling For Nginx
1. Select the right launch template
2. Select the VPC
3. Select both public subnets
4. Enable Application Load Balancer for the AutoScalingGroup (ASG)
5. Select the target group you created before
6. Ensure that you have health checks for both EC2 and ALB
7. The desired capacity is 2
8. Minimum capacity is 2
9. Maximum capacity is 4
10. Set scale out if CPU utilization reaches 90%
11. Ensure there is an SNS topic to send scaling notifications


### Set Up Compute Resources for Bastion

#### Provision the EC2 Instances for Bastion
1. Create an EC2 Instance based on CentOS Amazon Machine Image (AMI) per each Availability Zone in the same Region and same AZ where you created Nginx server
2. Ensure that it has the following software installed: python, ntp,net-tools,vim,wget,telnet,epel-release,htop.
3. Associate an Elastic IP with each of the Bastion EC2 Instances
4. Create an AMI out of the EC2 instance

#### Prepare Launch Template for Bastion (One Per Subnet)
1. Make use of the AMI to set up a launch template
2. Ensure the Instances are launched into a private subnet (private subnet 1 or 2)
3. Assign appropriate security group (The Bastion SG)
4. Configure Userdata to install mysql, ansible and git
```
#!/bin/bash
yum install -y mysql
yum install -y git tmux
yum install -y ansible
```

![image 10](https://user-images.githubusercontent.com/85270361/179700835-50fcd61a-cdaa-495d-b643-b1d2337a2d20.PNG)


#### Configure Target Groups
Go to Target Groups section and create a new target group.
1. Select Instances as the target type
2. Ensure the protocol is TCP on port 22
3. Register Bastion Instances as targets
4. Ensure that health check passes for the target group


#### Configure Autoscaling For Bastion
1. Select the right launch template
2. Select the VPC
3. Select both public subnets
4. Enable Application Load Balancer for the AutoScalingGroup (ASG)
5. Select the target group you created before
6. Ensure that you have health checks for both EC2 and ALB
7. The desired capacity is 2
8. Minimum capacity is 2
9. Maximum capacity is 4
10. Set scale out if CPU utilization reaches 90%
11. Ensure there is an SNS topic to send scaling notifications

### Set Up Compute Resources for Webservers

#### Provision the EC2 Instances for Webservers

Provision 2 separate launch templates for both the WordPress and Tooling websites

1. Create an EC2 Instance (Centos) each for WordPress and Tooling websites per Availability Zone (in the same Region).

2. Ensure that it has the following software installed: python, ntp,net-tools,vim,wget,telnet,epel-release,htop,php.

3. Create an AMI out of the EC2 instance

#### Prepare Launch Template For Webservers (One per subnet)

1. Make use of the AMI to set up a launch template
2. Ensure the Instances are launched into a private subnet (private subnet 1 or 2)
3. Assign appropriate security group (Webservers SG)
4. Configure Userdata to install httpd, wordpress, and configure rds to all work together. A sample of all userdata can be found in my repo(would link this at the bottom of this article) 


## STEP 3: TLS Certificates From Amazon Certificate Manager (ACM)

You will need TLS certificates to handle secured connectivity to your Application Load Balancers (ALB).

1. Navigate to AWS ACM
2. Request a public wildcard certificate for the domain name you registered in Freenom
4. Use DNS to validate the domain name. 
5. Tag the resource


![image 11](https://user-images.githubusercontent.com/85270361/179701129-153373b4-3d43-49bd-a157-0d58d106d024.PNG)

## STEP 4: CONFIGURE APPLICATION LOAD BALANCER (ALB)

#### Application Load Balancer To Route Traffic To NGINX
Nginx EC2 Instances will have configurations that accepts incoming traffic only from Load Balancers. This will allow us to offload SSL/TLS certificates on the ALB instead of Nginx. Therefore, Nginx will be able to perform faster since it will not require extra compute resources to valifate certificates for every request.

1. Create an Internet facing ALB
2. Ensure that it listens on HTTPS protocol (TCP port 443)
3. Ensure the ALB is created within the appropriate VPC | AZ | Subnets
4. Choose the Certificate from ACM
5. Select Security Group
6. Select Nginx Instances as the target group


#### Application Load Balancer To Route Traffic To Web Servers (this would be repeated for Wordpress webserver and tooling webservers)
Because of autoscaling, Nginx will not know about the new IP addresses, or the ones that get removed when there is a scale-out of the instances. Hence, Nginx will not know where to direct the traffic.

To solve this problem, we must use a load balancer. But this time, it will be an internal load balancer since the webservers are within a private subnet, and we do not want direct access to them.

1. Create an Internal ALB
2. Ensure that it listens on HTTPS protocol (TCP port 443)
3. Ensure the ALB is created within the appropriate VPC | AZ | Subnets
4. Choose the Certificate from ACM
5. Select Security Group
6. Select webserver Instances as the target group
7. Ensure that health check passes for the target group


![image 12](https://user-images.githubusercontent.com/85270361/179701483-5fb388f4-7675-485f-8041-0fdab91ce890.PNG)

### STEP 5:Setup EFS

1. Create an EFS filesystem
2. Create an EFS mount target per AZ in the VPC, associate it with both subnets dedicated for data layer

3. Associate the Security groups created earlier for data layer.
4. Create an EFS access point. (Give it a name and leave all other settings as default)

![image 13](https://user-images.githubusercontent.com/85270361/179701718-be80227c-478b-4ba2-96dc-bb7647b01f78.PNG)


### STEP 6: Setup RDS

We need to create a KMS to encrypt our database as a security measure.

To ensure that our databases are highly available and also have failover support in case one availability zone fails, we will configure a multi-AZ set up of RDS MySQL database instance.

1. Create a subnet group and add 2 private subnets (data Layer)
2. Create an RDS Instance for mysql 8.*.*
3. To satisfy our architectural diagram, select either Dev/Test or Production Sample Template. But to minimize AWS cost, selected the **Do not create a standby** instance option under Availability & durability sample template (The production template will enable Multi-AZ deployment)
4. Configure other settings accordingly (For test purposes, most of the default settings are good to go). 
5. Configure VPC and security (ensure the database is not available from the Internet)
6. Configure backups and retention
7. Encrypt the database using the KMS key created earlier
8. Enable CloudWatch monitoring and export Error and Slow Query logs (for production, also include Audit).
![image 14](https://user-images.githubusercontent.com/85270361/179702029-61eeb4b5-496c-4cc6-aae4-78e5b72bddc4.PNG)

## STEP 6: Configuring DNS with Route53

You can use either CNAME or alias records to achieve the same thing. But alias record has better functionality because it is a faster to resolve DNS record, and can coexist with other records on that name. I used A Records for this project. 

1. Create an alias record for the root domain and direct its traffic to the ALB DNS name.


![image 15](https://user-images.githubusercontent.com/85270361/179702368-329d4835-c866-4d8f-82e3-4a5a1dfb6c1b.PNG)

2. Create an alias record for tooling.<yourdomain>.com and direct its traffic to the ALB DNS name.


![image 16](https://user-images.githubusercontent.com/85270361/179702652-ca61c4f0-9ac8-4ec3-83e3-6428cae5bd36.PNG)


![image 17](https://user-images.githubusercontent.com/85270361/179702901-cc5f3af6-867d-47bc-800f-2d7c10079ecb.PNG)
