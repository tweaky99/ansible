# Welcome to the Ansible part of the workshop

## Intro
For Terraform there are currently 2 certified ansible collections:
1. [cloud.terraform](https://console.redhat.com/ansible/automation-hub/repo/published/cloud/terraform/) - Maintained by Red Hat. It uses the terrform cli to talk to terraform.
2. [hashicorp.terraform](https://console.redhat.com/ansible/automation-hub/repo/published/hashicorp/terraform/) - Maintained by HashiCorp
All future development is on the hashicorp.terraform collection. It is the collection for integration with HashiCorp Terraform Enterprise and Cloud and it is based on the provided API. This workshop uses this collection where possible and falls back to the older cloud.terraform collection where needed. 

## The basics
- Where is the Ansible Automation Server? -> [here](https://caap.fvz.ansible-labs.de)
- What is my username? -> The part before the @ of the email address you gave us
- What is my password? -> It's on the whiteboard in the workshop room

Ansible Automation Platform (AAP) has the notion of "Organizations". For this workshop an Organization named TechXchangeNL has been made and your account is part of that. Everything you do you will do within that organisation.

This workshop has been designed such that **you** will need to do most of the work, signifying the word "work" in workshop ;-) This means we only have the absolute basics set up and you need to build all the components to make everything work.

The division of work between Hashicorp Terraform Cloud and Ansible Automation Platform (as likely already explained to you) is:
- Terraform: Building up and changing infrastructure in the cloud, among which are RHEL10 servers
- AAP: Configure the RHEL10 servers to become a webserver serving a website

This means Terraform has all the credentials needed to do it's thing in the cloud of choice (AWS).
AAP will use the integrations with Terraform to be able to do it's thing on the provisioned infrastructure. AAP has no credentials for the cloud!

So what are the basics we have set up for you:
1. A machine credential named RHEL. You use this machine credential to be able to run your playbooks on the providioned servers.
2. A Custom Credential _Type_ called "Hashicorp Terraform Cloud". You use this later to create your credential. See [here](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/getting_started_with_hashicorp_and_ansible_automation_platform/terraform-product#creating-custom-credential-type) for details.
3. An Inventory called "local" with the host "localhost" for api based automations.
4. A token to be able to do stuff in Hashicorp Terraform Cloud. This token is available at: <TBC>
5. An Execution Environment called "ee-tech-x-change-nl" in AAP that provides all the collections and dependencies you need in this workshop.

## Building Blocks
You need to create some building blocks in AAP for this workshop. This document explains what you need to make. When you are done, go back to the README.md in the workshop repo.

note: wherever you can and/or need to specify an Organisation, choose "TechXchangeNL'.

### Project
You need to create a project in AAP. The project is your repository with playbooks.
- The location of the git repo with the platybooks for this workshop: [here](https://github.com/TechXchangeNL/ansible.git)

### Credentials
Apart from the already available machine credential, you need a few more..

- A Credential to be able to communicate with Hashicorp Terraform Cloud. Use Credential Type "Hashicorp Terraform Cloud" and the provided token.
- A Credential to be able to sync the Terraform State File that will be used for the inventory source. Choose the credential type "Terraform backend configuration". In the backend configuration field enter the following:

```text
hostname = "app.terraform.io"  
organization = "TechXchangeNL"  
token = "YOURTOKENHERE"  
workspaces { name = "YOURWORKSPACE" }  
```
For token, enter the token provided
For workspace anter the workspace you made in Terraform (you did...right?)

### Inventories
Create an inventory called "TechXchangeNL" and add a dynamic inventory source to it named "Terraform". This source needs a special plugin and some configuration to do the magic of syncing the statefile. The plugin is in the provided execution environment and the config that you need to give in the _Source Variables_ are:
```text
plugin: cloud.terraform.terraform_state
backend_type: remote
compose:
  ansible_host: public_ip
hostnames:
  - tag:Name
keyed_groups:
  - prefix: os
    key: tags.OS
  - prefix: role
    key: tags.Role
```
Also, you need the Terraform Backend Configuration Credential you made as the credential for this source. You can test it by syncing the source manually. Finally you need to enable _update on launch_ on the inventory source.

### Job Templates
Now that you have the basics set up (project, credential, inventory), you can define job templates in AAP. As you can see in the repository where this text lives, there are 4 playbooks:
- deploy_servers.yml This playbook will apply a plan in Hashicorp Terraform Cloud.
- update_server.yml. This playbook will update RHEL servers
- deploy_webserver.yml. This playbook will deploy a webserver (apache)
- deploy_website.yml. This playbook will dpeloy a website
Have a look at the playbooks in this repo to get a sense of what they do.

Create a Job Template for each of these playbooks.
For the deploy_servers playbook:
- Use the provided "local" inventory. For all others the "TechXchangeNL" inventory.
- Use the "Hashicorp Terraform Cloud" credential you made.
- Add an extra var "workspace" with as the value the name of your workspace

For the other playbooks:
- Use the TechXchangeNL inventory
- Use the TechXchangeNL machine credential

### Workflows
Having job templates (automation building blocks) we create two workflows:
1. A workflow (name suggestion: "Deploy Web App") that runs the following playbooks in that order:
   1. sync inventory source Terraform
   2. update_server
   3. deploy_webserver
   4. deploy_website
2. A workflow (name suggestion: "Deploy Full Web App") that run the following in that specific oprder:
   1. deploy_servers
   3. the proviously created workflow

### API Token
Part of the workshop is showing how you can run stuff _in_ AAP _from_ HashiCorp Terraform Cloud. For this, you need to provide a token from AAP to your HashiCorp Terraform Cloud workspace. You can create a token yourself using _API token_ under _Access Management_ in the menu. Choose write access. Copy/Paste the token somewhere, because it will only be shown once! HashiCorp Terraform Cloud will need both the username and the token.
