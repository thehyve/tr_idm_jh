# Deployment of jupyterhub for IDM

## Description

The goal is to simplify access for CLI tools of IDM.

## Used tools

We use deploy a [JupyterHub](https://jupyter.org/hub) instance to simplify access for CLI tools of IDM.

This project deploys the IDM to the [Hetzner Cloud](https://www.hetzner.com/cloud), but it should be possible to use this project as inspiration for the development of a project deploying to another cloud.

We provision our servers using [Terraform](https://www.terraform.io/), and if it was decided to go with another cloud, it is possible to use an orchestration tool from that cloud such as [AWS CloudFormation](https://aws.amazon.com/cloudformation/) or [Openstack Heat](https://wiki.openstack.org/wiki/Heat) or any other preferred one.

We use [FreeIPA](https://www.freeipa.org/page/Main_Page "The Open Source Identity Management Solution") as our IDM solution.

We use [CentOS](https://centos.org/) 8 Linux distribution as a base operating system, but it should be possible to use any distribution that is able to run Podman, uses systemd, and if its kernel has the support for IPvlans and network namespaces.
But the current implementaion containes some steps that are specific for CentOS 8.
If it is decided to go with other Linux distribution, those steps should be modified.

After servers are provisioned [Ansible](https://www.ansible.com/) is used for the deployment and the configuration of a JupyterHub instance.

And we use [ansible_freeipa](https://galaxy.ansible.com/freeipa/ansible_freeipa) Ansible collection to make the host idm client.

## Preparation

### External dependencies

#### Terraform configuration

This project uses Terraform for infrastructure provisioning and uses Terraform remote backend for keeping the state of infrastructure for collaborative work.

To be able to use the remote backend we need to register (or have already registered) __*organizaton_name*__ organization at [Terraform Cloud](https://app.terraform.io/).
Also we need to create __*workspace_name*__ workspace for this deployment.
This workspace should have __Local__ Execution mode in its settings, so it will be used only to store the state of our deployment.
This requierements comes from the fact that there is a __local-exec__ provisioner in the projects Terraform code, that generates Ansible inventory file to be used in the next step.

#### Hetzner cloud configuration

It is assumed that there is an account Hetzner, so we are allowable to create new entities in Hetzner Cloud.
We should create a project that will include infrastructure for this project.

### Local configuration

#### Terraform

##### Terraform CLI installation

Use this [manual](https://learn.hashicorp.com/terraform/getting-started/install.html) to install Terraform CLI tool.

##### Terraform access configuration

In order to be able to access Terraform Cloud from Terraform CLI we need generate an access token in "User settings" at Terraform Cloud, and put it into the `~/.terraformrc` file:
```
credentials "app.terraform.io" {
  token = "xxxxxx.atlasv1.zzzzzzzzzzzzz"
}
```

##### Terraform backend initialisation:

In order to initialise Terraform backend there shoud be created `backend.hcl` file in the root directory of this project with the following content:
```
# backend.hcl
hostname = "app.terraform.io"
organization = "<organizaton_name>"
workspaces { name = "<workspace_name>" }
```
Replace words in `< >` with your __*organizaton_name*__ and __*workspace_name*__.

Although this file containes no sensitive information it is included into `.gitignore` file.

Run the following command for terraform initialisation:
```bash
terraform init -backend-config=backend.hcl
```

#### Hetzner

##### Hetzner CLI installation

Use this [manual](https://community.hetzner.com/tutorials/howto-hcloud-cli) for CLI installation and configuration.

#### Ansible

##### Ansible installation

Use this [manual](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) for Ansible installation.

##### Installation of ansible_freeipa collection

Follow [instructions](https://galaxy.ansible.com/freeipa/ansible_freeipa) to install ansible_freeipa collection.

## Basement

### Disclosure

The creation of some base entities lies outside of the code of this project for certain reasons.

### Mapping definiton

We need to define some values and map them to each other.
* `host` - short hostname that will be assigned to a server running a JupyterHub instance.
* `location` - the name of a location of Data Center where the `host` is running.

Example values for a single node:
```
location: nbg1
host: jupyter
```
To prevent an ocasional selection of an entity we define additional key-value map for all objects:
```
object: freeipa
```
The code containes some default mappings, but this can be easily changed by assigning different values to variables.
See [configuration](#configuration) chapter.

### IP addresses and DNS records

#### Explanation

DNS records for this deployment will be automaticly created during the procedure of idm client instalation in DNS zone managed by IDM.

### Data volumes

#### Explanation

This setup relies on a external volume to store users home directories.
We keep the process of volumes creation outside of the projects code to prevent accedentally data deletion.

That also allows us to easily recover in case of host failure.

#### Volumes creation

1. Create a volume using `hcloud` utility (*example*):
   ```bash
   hcloud volume create --location nbg1 --name jupyter --size 128
   ```
2. Add labels to a volume (*example*):
   ```bash
   hcloud volume add-label VOLUME_ID host=jupyter
   hcloud volume add-label VOLUME_ID object=freeipa
   ```

## Deployment

### Configuration

It is important to be sure that we went through all previous step and have `backend.hcl` file created and Terraform initialisated.

We need to define values for some variables:
* `hcloud_token` - __required__;
* `server` - optional, a mapping of a server name to the server location (data centers). The default mapping defined in the code. Be aware this mapping should be relevant for labels of a volume;
* `server_type` - optional, the default value defined in the code. Check Hetzner for all possible values;
* `ssh_key` - __required__. An ID of your public key previously uploaded to Hetzner cloud;
* `ssh_key_private` - __required__. A string with a path to the private ssh-key;
* `remote_user` - optional. The default user for OS images in Hetzner cloud is `root`;
* `server_image` - optional. A name of OS image, the default value defined in the code;
* `domain` - __required__. A name for domain zone that is managed by IDM system;

There are several way to do that:

* Use environment variables:
   It is possible to create executalbe `vars.sh` file with the following content:
   ```bash
   #!/usr/bin/env bash
   export HCLOUD_TOKEN=...
   export TF_VAR_ssh_key=...
   export TF_VAR_ssh_key_private=...
   export TF_VAR_domain=...
   ```
* Create `terraform.auto.tfvars` file containning variable in Terraform format:
   ```
   ssh_key_private = "/home/username/.ssh/id_ed25519"
   remote_user = "..."
   server_image = "..."
   ssh_key = "..."
   domain = "..."
   hcloud_token = "..."
   ```

Both of these files are mentioned in `.gitignore` file.

### Resource provisioning

After we are done with previous steps, we can:

1. Verify Terraform code with the following command:
   ```bash
   terraform validate
   ```
2. Run the following command to provision servers:
   ```bash
   terraform run
   ```

### Configuring and running FreeIPA services

At the last step Terraform will create (update) `inventory.yml` file from `inventory.template`.
This file defines values for all needed variables except 2 most important ones:
* a username of an account with priveleges sufficient to add a host to IDM as a client;
* a password for that user.

These 2 values shoud be kept in secret.

There several way to pass these values to our ansible playbook:
* passing them as `--extra-vars` parameter to `ansible-playbook` command, if we are sure that our CLI history is a safe place;

* creating a file with variables, protected by `ansible-vault` with following command:
   ```bash
   ansible-vault edit varsafe.yml
   ```
   with a following content:
   ```yaml
   ---
   ipaadmin_principal: "username"
   ipaadmin_password: "password"
   ...
   ```

Here is an example of ansible command that we can use for configuring and running a JupyterHube service:
```bash
ansible-playbook -i inventory.yml --extra-vars "@varsafe.yml" --ask-vault-pass main.yml
```
