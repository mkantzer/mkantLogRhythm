# mkantLogRhythm

## Introduction
This codebase was created for Michael Kantzer's evaluation with LogRhythm. The task was to automate the creation of a GlusterFS storage cluster with single node failure fault tolerance with no data loss. Automation should start with the entry of a single command and should use modern tools appropriate for the task; these requirements have been met with the use of the following:
  * Amazon Web Services for infrastructure
  * Ansible for automation
## Requirements Before Beginning

This solution is built to run on Amazon Web Services (AWS); As such, an AWS account is required. Due to the relatively small nature of the solution, a Free Teir account should be sufficient. Note: Leaving multiple instances running beyond the end of the demonstration may exceed the limits imposed by AWS Free Tier, whcih may result in additional costs. Please be sure to clean up your environment after the evaluation. 

Various components (primarily, the disk snapshot I have created and shared) are hosted in the AWS us-east-1 region. As the solution requires access to these resources, the evaluation environment must be hosted within AWS us-east-1. 

Additionally, an initial environmental setup is required. Please follow the instructions below (documented in the Installation section) to perform this setup. Setup assumes knowledge of the AWS web console, experience with SSH tools, and proficiency in using a command line interface. 

If you have any questions on a step, please feel free to contact me. 


## Initial Setup 

**NOTE: Full deployment must occur within the us-east-1 (N. Virginia) AWS region** due to resource availability. 

AWS Configuration
* Create New VPC  
Specifications:  
	* IPv4 CIDR block: 10.0.0.0/16  
	* Enable DNS hostnames  
* Create 2 New Subnets  
	* When assigning Availability Zones there are no specific zones that need to be selected, however the zone for Subnet 1 must be different than the zone for Subnet 2.
	* Do not forget to document subnet-IDs!
	
	Specifications - Subnet 1:  
	* VPC: New VPC created above
	* IPv4 CIDR Block: 10.0.0.0/20
	* Assign Availabily Zone
	* Enable Auto-assign public IPv4 addresses

	Specifications - Subnet 2:  

	* VPC: New VPC created above
	* IPv4 CIDR Block: 10.0.16.0/20
	* Assign Availabily Zone
	* Enable Auto-assign public IPv4 addresses

* Create new Network Access Control List  
Specifications:
   	* VPC: New VPC created above
	* Subnets: Both new Subnets created above
	* Rules:
		* for both inbound and outbound:

		| Rule #  | Type | Protocol | Port Range | Source/Destination | Allow/Deny |
		| ------------- | ------------- | ------------- | ------------- | ------------- | ------------- |
		| 100 | ALL Traffic | ALL | ALL | 0.0.0.0/0 | ALLOW |
		| * | ALL Traffic | ALL | ALL | 0.0.0.0/0 | DENY |
		
		* Please see "Known Issues" for justification for the excessive permissiveness

* Create new Internet Gateway  
Specifications:  
	* VPC: New VPC created above

* Create New Route Table  
Specifications:
	* Subnet Associations: Both new Subnets created above
	* Routes:  
	
		| Destination | Target |
		| --- | --- |
		| 0.0.0.0/0 | {New IGW-ID}
		* Please see "Known Issues" for justification for the excessive permissiveness

* Create New Security Group  
	* Note down the security group ID  
Specifications:  
	* VPC: New VPC created above
	* Rules: 

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

* Create New IAM role  
Specifications:  
	* Name: Ansible
	* Service Role: EC2
	* Permissions: Poweruser

* Create New IAM User  
Specifications:
	* Name:  Ansible
	* Access Type: Programatic
	* Permissions: Poweruser (from existing policies)
	* **Make sure you note both the Access Key ID and the Secret Key for later use**
* Create New Key Pair
    * **Make sure you save the .pem it will download. This will be needed to access all instances created**
    * Note down name of key

* Create New EC2 instance for Ansible
    * Select free-tier Ubuntu 14.04 LTS, ami-49c9295f 
    	* (if this image is not available, copy new value for later use)
    * Select t2.micro
    * Select the new VPC
    * Select Auto-assign public IP
    * Select IAM Role: Ansible
    * Select Shutdown behavior: stop
    * Select 8 GB GP2 storage, delete on termination
    	* This should be the standard, and auto-populated
    * Enter Tag: Name | Ansible
    * Select the new Security group
    * Select the new key pair when launching

* SSH into the EC2 instance, using the downlaoded .pem, and connecting to ubuntu@{public DNS of instance}
    * If connecting from windows/PuTTY, you may need to use puttygen to convert the .pem to a ppk
* Run the following commands:
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
		
* Edit environmental variables:

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

Following Initial Setup, the environment is fully configured, and is ready to execute the deployment playbook:

`ansible-playbook master.yml`

Allow the playbook to run completely. 

## Verification

Check your AWS EC2 instances console. There should be 4 new instances, 2 named GlusterCluster and 2 named GlusterClient. 

SSH into the 2 Client machines, navigate to `/mnt/gluster`, and then create a file in one with `sudo touch test.txt`
Now, `ls` in both, and you should see that file replicated to the second instance.  

You can stop (not terminate) either of the instances labeled GlusterCluster, and the replication will still work. 
Please note that you should fully re-start whichever is down prior to stopping the other one.  


## Solution Description

This project uses Ansible v2.2.2.0, hosted in Amazon Cloud Services, to deploy two GlusterFS nodes, configure them into a cluster, configure and launch a redundant volume on that cluster, and then mount that volume on two clients.
The networking configuration places one node and one client on each of two sub-nets, which are hosted in different availability zones for increased reliability; if one availability zone goes down, the other is still accessible. 

Using two GlusterFS nodes with "replica 2" set on them achieves the single-fault-tolerant requirement; either one can be lost without losing any data. 

Tool Choices:
  * AWS was chosen for speed, ease of use, familiarity, and ability to integrate automation easily.
  * Ansible was chosen because of its notable ease of use, speed of development, use of Python and YAML syntax, and wide variety of built-in modules ("batteries included"). This extended all the way down to a glusterFS volume management module. Additionally, it was extremely easy to connect to AWS for dynamic inventory management. Finally, Ansible is known for its idempotency, which I have done my best to maintain; it checks for the current status before making changes, only when they are required to return a system to the desired configuration.  

## Known Issues and Breaches of Best Practices

  * The IAM role used by Ansible has a larger amount of permissions than strictly required. It would be better to limit these to the minimum required. 
  * My AWS permissions are extremely loose, especially in the Network Access Control Lists, route tables, and the Security Groups. These should be locked down much further, allowing connections to and from only the IP addresses that require them, and restricting traffic types. However, for this proof of concept, flexibility was considered more important than direct security, as no data would actually be housed within the servers.
  * Boto is currently receiving access keys via environment variables. While this does work, there are cleaner and more permanent ways to accomplish this (namely via roles)
  * Source control/playbooks/scripts are all stored in /etc/ansible. This is so that I could correctly manage the .cfg and .ini files, as well as proving a central place to hold everything. 
  * There is a 30 second pause in the middle of execution to allow Boto/ec2.py to re-cache the inventory. This, in conjunction with decreasing the ec2.ini value for the time to keep a cache to 25 seconds (from 300) is an inelegant solution to the problem of re-initializing the cache after adding new instances, as well as allowing for enough time for them to become available. A better solution would most likely be along the lines of a looping structure that exits when it detects the new instances (or after a set time)
  * If a Cluster node fails (for example, is terminated from the AWS console), re-running the playbook does not add a new node to the cluster. 
  * ignore_errors has been set to true when creating the gluster volume. This is due to a bug in the module, where it attempts to execute "gluster volume add-brick" against each server in the cluster, when it only needs to execute against a single one. As a result, on the second node, it tries to add a brick that is already added, and runs into an error. The volume was still created, however, so I am treating this as a success condition. Please note that this occurs even though I have flagged run_once in the task, so that it still only runs the command on a single server. 
  * Security has not been configured on GlusterFS; anyone can currently mount the volume if they know where to point. This is because of the previously mentioned bug in the gluster_volume module
  * I am currently deploying the extra drive to the GlusterFS nodes using a public snapshot I have created. This is due to requiring a specifically formatted partition. While this is possible in Ansible 2.3 using the new "parted" module, my current solution utilizes v 2.2.2.0, the latest available for Ubuntu. 


## Potential Improvements

  * Deploy the Ansible Master from a pre-configured snapshot/AMI. This was not done for the current implementation because of sharing requirements. 
  * This solution only has 2 nodes connected to the replicated volume. In a production environment, it would be best to use 3 (spread between 3 availability zones) for increased redundancy
