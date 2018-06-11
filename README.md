# Packer AMI Builder

## Overview

This project is used to deploy CodePipeline/CodeBuild into AWS using Cloudformation which will run and deploy a packer template to build private AMI's.

The codepipeline will use webhook to clone this repository to an S3 bucket. It will then deploy the content to a CodeBuild Container.

Please note that this should only be used for testing and development.


### Prerequisites

1. Ensure you have a GitHub Token for this repository
2. Have GitHub Desktop installed
3. Access to AWS with enough privelages to deploy Cloudformation templates


## Getting Started

You will need to clone this repository locally.

Log in to AWS and select Services, Cloudformation, then Create Stack.

Choose file and browse to the cloned repository.

Go into the 'Cloudformation' folder, then select the folder containing the template you require, for example 'zabbix-base-test', then select the json file 'zabbix-base-test.json'.

Hit Next.

Then add the following parameters:

- Stack Name
- Bucket Name
- GitHub Branch
- GitHub Owner
- GitHub Repo
- GitHub token
- PathToBuildSpec

Hit next and next again.

Check your input details and acknowledge by ticking the box.

Hit Create.

## Deployment

Once deployed, the Cloudformation template will create the following services:

#### S3: 
This will be used to store data cloned from Github and for storing artifact outputs from CodePipeline

#### CodePipeline:
This will link to this GitHub repository via a webhook, it will trigger a deployment whenever there are any changes to this repository (you will need to add which branch this occurs on)

#### CodeBuild: 
This will run through the Build.yaml file which can be found in the CodeBuild/zabbix-base-test/ folder. You can specify different locations for this in the Cloudformation template parameter 'PathToBuildSpec'

## A run through of the CodeBuild steps 
Within the Build.yaml file, this will run through what happens on the CodeBuild container in 3 phases:

### Pre-Build
This phase will download packages needed to build the private AMI: Packer and jq (used to add variable strings to the credentials json file)
Once thats downloaded, it will a run a packer validation on the chosen template.


### Build
This phase starts the process of building the AMI
First it will need to authenticate in order to run Packer in AWS. To do this, it will utilise the following command to export the services credentnials:
aws_credentials.json http://169.254.170.2/$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI

This is required as Packer cannot use IAM roles, it need access key and secret key to work.

Once authentication and region and configured, it will build the packer template

## A Run through of the Packer Template
The variables are captured from the local machines settings, in this case it is the CodeBuild Container. The variables for the AWS authorisation and region were set in the BuildSpec.yaml template.

The next step is the 'Builders' Process:

This sets the access Key and secret key Id's and region, you can set the name of the AMI here, what type of instance, the AMI you are using to build from.

After this has been set, the next step in the template is the 'Provisioners' process:

This part runs through all the steps to install what is required for the AMI (example, installing Ansible to configure all other applications such as Zabbix, Apache.. etc)

## Ansible
The Ansible folder currently contains 1 Playbook which is used to install and configure Zabbix Agent.

After CodeBuild installs Ansible, it then runs through the Packer template, as mentioned it uses Ansible playbooks to install additional applications and configures them.

### Overview of Zabbix PLaybook

The Packer template runs the followuing file (currently this is in Ansible/Playbooks/zabbix-agent/deploy-zabbix-agent.yaml) which initiates the playbook configuration.

This contains hosts (should be localhost if using with packer), then Sudo commands and the Role (currently this is zabbix-agent)

Within the zabbix-agent role, you should have 3 folders: Handlers, Tasks, Templates, each has a main.yaml file.

The main configuration is lcoated in the Tasks folder, this runs through all installs, creating files/directories etc

The Template folders contains files which can be used to copy over to the remote machine, such as conf files.

The Handlers folder contains Handler functions, so when a changfe is made in a task, it will clal a handler (for example restart a service).


## Summary

This is currently in a POC stage, the idea is to create AMI's for each environment which can be used in autoscaling groups. This will help speed up deployments and when scaling out.

Once these are deployed, Ansible wil be used for desired state configuration.

This build process should be run every 3 months regardless of any changes to the build, as this will create an AMI which has latest patches from AWS.

The AMI names should have a time stamp on. We can then use Lambda to check current AMI's deployed, for UAT we can automate to update to new AMI, for Production we can create a notification to do this manually (at least for now).
