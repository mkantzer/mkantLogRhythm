# mkantLogRhythm

## Introduction
This codebase was created for Michael Kantzer's evaluation with LogRhythm. The task was to automate the creation of a GlusterFS storage cluster with single node failure fault tolerance, with no data loss. Automation should start with the entry of a single command, and should use modern tools appropriate for the task
These requirements have been met with the use of:
  * Amazon Web Services for infrastructure
  * Ansible for automation
## Requirements Before Begininng

This solution is built to run on Amazon Web Services. As such, an AWS account is required. Due to the relatively small nature of the solution, it should stay within the allotment for Free Teir usage. However, leaving multiple instances running past the end of the demonstration will exceed this limit, and could incur costs. Please be sure to clean up your environment after the evaluation. 

Various components (namely, a disk snapshot that I have created and shared) are hosted in the AWS us-east-1 region. As the solution requires access to these resources, the evaluation environment must be hosted within AWS us-east-1. 

Additionally, initial environmental setup is required. This is relatively straightforward, and is documented below, under Installation. Please be sure to follow the instructions carefully. 

Setup assumes knowledge of the AWS webconsole, proficiency with command line, and knowlege of how to use ssh tools. 

If you have any questions on a step, please feel free to contact me. 


## Installation

**PLEASE NOTE: Full deployment must occur within the us-east-1 (N. Virginia) AWS region** due to resource availability. 

VPC setup:
1. Create New VPC
    1. 10.0.0.0/16 for IPv4 CIDR block
    2. Enable DNS hostnames
2. Create 2 Subnets
	1. Attached to the new VPC
	2. 10.0.0.0/20 for IPv4 CIDR block on the first
	3. 10.0.16.0/20 for IPv4 CIDR block on the second
	4. Assign them to different availability zones
	5. Ensure they auto-assign public IPv4 addresses 
		* you may need to change this in subnet actions
	6. Note down the subnet ids
3. Network Access Control Lists
	1. Create new NACL, and connect it to the new VPC
	2. Make sure both subnets are connected to the new NACL
	3. Set the rules:
		* for both inbound and outbound, create a new rule, number 100, for all traffic types, from all protocols, on all port ranges, for source 0.0.0.0/0, set to ALLOW
		* Please see Known Issues for justification for the excessive permissiveness
4. Create Internet Gateway
	* Attach it to the new VPC
5. Create Route Table
	1. Connect both new subnets to the table
	2. Under routes, add 0.0.0.0/0 and your IGW name
		* Please see Known Issues for justification for the excessive permissiveness
6. Create Security Group
	1. Associate it with the new VPC
	2. Note down the security group ID  
	3. Rules: 
		* For both inbound and outbound

		| Type  | Protocol | Port Range | Source |
		| ------------- | ------------- | ------------- | ------------- |
		| SSH | TCP | 22 | 0.0.0.0/0 |
		| SSH | TCP | 22 | ::/0 |
		| HTTP | TCP | 80 | 0.0.0.0/0 |
		| HTTP | TCP | 80 | ::/0 |
		| HTTPS | TCP | 443 | 0.0.0.0/0 |
		| HTTPS | TCP | 443 | ::/0 |
		| ALL traffic | ALL | ALL | {id for this security group} |
		* Outbound should have an additional:
		
		| Type  | Protocol | Port Range | Source |
		| ------------- | ------------- | ------------- | ------------- |
		| ALL traffic | ALL | ALL | ::/0 |

7. Create IAM role
	* Named Ansible
	* EC2 service role
	* Grant Poweruser Access
8. Create IAM User
	* Named Ansible
	* Programatic Access
	* Poweruser Permissions (from existing policies)
	* **Make sure you note both the Access Key ID and the Secret Key for later use**
9. Create new Key Pair
	* **Make sure you save the .pem it will download. This will be needed to access all instances created**
	* Note down name of key

10. Create Ansible Master
	1. Launch EC2 instance:
		* free-tier Ubuntu 14.04 LTS, ami-49c9295f 
			*(if this image is not available, copy new value for later use)
		* t2.micro
		* Connected to the new VPC
		* Auto-assign public IP
		* IAM Role: Ansible
		* Shutdown behavior: stop
		* 8 GB GP2 storage, delete on termination
			* This should be the standard, and auto-populated
		* Tag: Name | Ansible
		* Security group: The new one
		* When launching, be sure to use the new key pair
	2. SSH into the EC2 instance, using the downlaoded .pem, and connecting to ubuntu@{public DNS of instance}
		* If connecting from windows/PuTTY, you may need to use puttygen to convert the .pem to a ppk
	3. Run the following commands:
		```
		sudo apt-get update
		sudo apt-get -y install git
		sudo git clone https://github.com/mkantzer/mkantLogRhythm /etc/ansible
		sudo apt-get -y install software-properties-common
		sudo apt-add-repository -y ppa:ansible/ansible
		sudo apt-get update
		sudo apt-get install -y ansible
			* You may be prompted about repeated files. if so, keep current. 
			* I have made changes to some cfg files. 
			* Use option 'N'
		sudo apt-get install -y python-pip
		sudo pip install -U boto
		touch ~/.ssh/{name_of_new_key}.pem
		chmod 700 ~/.ssh/{name_of_new_key}.pem
		vi ~/.ssh/{name_of_new_key}.pem
			* Paste contents of your .pem into this file
			* save with :wq
		ssh-agent bash
		ssh-add ~/.ssh/{name_of_new_key}.pem
		sudo chmod +x /etc/ansible/ec2.py
		export AWS_ACCESS_KEY_ID='Access_Key'
			* replace Access_Key with the one noted earlier
		export AWS_SECRET_ACCESS_KEY='Secret_Key'
			* replace Secret_Key with the one noted earlier
		export ANSIBLE_HOSTS=/etc/ansible/ec2.py
		cd /etc/ansible
		ansible all -m ping
			* This should return a green success (and sometimes a purple warning)
		```
		
	4. Edit environmental variables:

		`sudo vi group_vars/all`
		
		| Variable | Use |
		| --- | --- |
		| AWSkey_name | Key pair name created earlier, used in deploying and accessing instances |
		| AWSgroup_id | Security Group ID |
		| AWSregion | Region name. This should be us-east-1 |
		| AWSvpc_subnet_id | IDs for subnets created earlier |
		| AWSimage_id | Image to use in deploying instances. Please use ami-49c9295f unless it was unavailable earlier; solution is configured for ubuntu 14.04 |
				
Note: If you ever disconnect from your SSH session, you will need to re-initialize:

```
ssh-agent bash 
ssh-add ~/.ssh/{key name}.pem 
export AWS_ACCESS_KEY_ID='{key}' 
export AWS_SECRET_ACCESS_KEY='{secret}' 
export ANSIBLE_HOSTS=/etc/ansible/ec2.py
cd /etc/ansible
```		
		
## Execution

At this point, we are fully configured, and are ready to execute the deployment playbook:

`ansible-playbook master.yml`

Allow the playbook to run completely. 

## Verification

Check your AWS EC2 instances console. There should be 4 new instances, 2 named GlusterCluster and 2 named GlusterClient. 

SSH into the 2 Client machines, navigate to `/mnt/gluster`, and then create a file in one with `sudo touch test.txt`
Now, `ls` in both, and you should see that file replicated to the second instance.  

You can stop (not terminate) either of the instances labeled GlusterCluster, and the replication will still work. 
Please note that you should re-start whichever is down before stopping the other one.  


## Solution Description

This project uses Ansible v2.2.2.0, hosted in Amazon Cloud Services, to deploy two GlusterFS nodes, configure them into a cluster, configure and launch a redunant volume on that cluster, and then mount that volume on two clients.
The networking configuration places one node and one client on each of two sub-nets, which are hosted in different availability zones for increased reliability; if one availabilty zone goes down, the other is still accessable. 

Using two GlusterFS nodes with "replica 2" set on them achieves the single-fault-tolerent requirement; either one can be lost without losing any data. 



Tool Choices:
  * AWS was chosen for speed, ease of use, familiarity, and ability to integrate automation easily.
  * Ansible was chosen because of its notabel ease of use, speed of development, use of Python and YAML syntax, and wide variety of built-in modules ("batteries included"). This extended all the way down to a glusterFS volume management module. Additionally, it was extreemly easy to connect to AWS for dynamic inventory management. Finally, ansible is known for its idempotency, which I have done my best to maintain; it checks for the current status before making changes, only when they are required to return a system to the desired configuration.  
	


## Known Issues and Breachs of Best Practices

  * The IAM role used by Ansible has a larger amount of permissions than strictly required. It would be better to limit these to the minimum required. 
  * My AWS permissions are extreemly loose, especially in the Network Access Control Lists, route tables, and the Security Groups. These should be locked down much further, allowing conenctions to and from only the IP addresses that require them, and restriciting traffic types. However, for this proof of concept, flexability was considered more important than direct security, as no data would actually be housed wihtin the servers.
  * bota is currently receiving access keys via environment variables. While this does work, there are cleaner and more permanant ways to accomplish this (namely via roles)
  * Source control/playbooks/scripts are all stored in etc/ansible. This is so that I could correctly manage the .cfg and .ini files, as well as proving a central place to hold everything. 
  * There is a 30 second pause in the middle of execution to allow boto/ec2.py to re-cache the inventory. This, in conjunction with decreasing the ec2.ini value for the time to keep a cahce to 25 seconds (from 300) is an in-elegent solution to the problem of re-initalizing the cache after adding new instances, as well as allowing for enough time for them to become avaialable. A better solution would most likely be along the lines of a looping structure that exits when it detects the new instances (or after a set time)
  * If a Cluster node fails (for example, is terminated from the AWS console), re-runnign the playbook does not add a new node to the cluster. 
  * ignore_errors has been set to true when creating the gluster volume. This is due to a bug in the module, where it attempts to execute "gluster volume add-brick" against each server in the cluster, when it only needs to execute against a single one. As a result, on the second node, it tries to add a brick that is already added, and runs into an error. The volume was still created, however, so I am treating this as a success condition. Please note that this occurs even though I have flagged run_once in the task, so that it still only runs the command on a single server. 
  * Security has not been configured on GlusterFS; anyone can currently mount the volume if they know where to point. This is because of the previously mentioned bug in the gluster_volume module
  * I am currently deploying the extra drive to the GlusterFS nodes using a public snapshot I have created. This is due to requiring a specificly formated partition. While this is possible in ansible 2.3 using the new "parted" module, my current solution utilizes v 2.2.2.0, the latest available for Ubuntu. 


## Potential Improvements

  * Deploy the Ansible Master from a pre-configured snapshot/AMI. This was not done for the current implementation because of sharing requirements. 
  * This solution only has 2 nodes connected to the replicated volume. In a produciton environment, it would be best to use 3 (spread between 3 availability zones) for increased redundency
