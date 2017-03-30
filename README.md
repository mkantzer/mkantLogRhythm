# mkantLogRhythm

Introduction
=======

This codebase was created for Michael Kantzer's evaluation with LogRhythm. The task was to automate the creation of a GlusterFS storage cluster with single node failure fault tolerance, with no data loss. Automation should start with the entry of a single command, and should use modern tools appropriate for the task

These requirements have been met with the use of:

  * Amazon Web Services for infrastructure
  * Ansible for automation


Requirements Before Begininng
=================

This solution is built to run on Amazon Web Services. As such, an AWS account is required. Due to the relatively small nature of the solution, it should stay within the allotment for Free Teir usage. However, leaving multiple instances running past the end of the demonstration will exceed this limit, and could incur costs. Please be sure to clean up your environment after the evaluation. 

Various components (namely, a disk snapshot that I have created and shared) are hosted in the AWS us-east-1 region. As the solution requires access to these resources, the evaluation environment must be hosted within AWS us-east-1. 

Additionally, initial environmental setup is required. This is relatively straightforward, and is documented below, under Installation. Please be sure to follow the instructions carefully. 

Setup assumes knowledge of the AWS webconsole, proficiency with command line, and knowlege of how to use ssh tools. 

If you have any questions on a step, please feel free to contact me. 


Installation
============

**PLEASE NOTE: Full deployment must occur within the us-east-1 (N. Virginia) AWS region ** due to resource availability. 

VPC setup:
1. Create New VPC
  1. 10.0.0.0/16 for IPv4 CIDR block
  2. enable DNS hostnames

2. Create Subnets
  1. 



Execution
==================






Solution Description
======================








 * bullited
 * list
 * of
 * things
 `to mark it as a box`
  [releases.ansible.com](https://releases.ansible.com/ansible)
 
Execution
===========


Known Issues and Breachs of Best Practices
=======
	

  * if a Cluster node fails (for example, is terminated from the AWS console), re-runnign the playbook does not add a new node to the cluster. 
  *ignore_errors has been set to true when creating the gluster volume. This is due to a bug in the module, where it attempts to execute "gluster volume add-brick" against each server in the cluster, when it only needs to execute against a single one. As a result, on the second node, it tries to add a brick that is already added, and runs into an error. The volume was still created, however, so I am treating this as a success condition. Please note that this occurs even though I have flagged run_once in the task, so that it still only runs the command on a single server. 
  * Security has not been configured on GlusterFS; anyone can currently mount the volume if they know where to point. This is because of the previously mentioned bug in the gluster_volume module


Potential Improvements
=======