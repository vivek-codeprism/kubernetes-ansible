# Summary
The kubernetes-ansible repo is designed to create a VPC and self contained kubernetes cluster in AWS.

## Variables
Throughout this document, where you see ${VPCNAME}, replace it with the proper name such as 'k8s1'.

## Configuration Files
You will need to create the assorted vpc specific configuration files under vars/${VPCNAME}/.  They will need to be tailored to the specific vpc being created.  Additionally, the creation of a running vpc will generate files within the vars/${VPCNAME}/ directory specific to the created vpc.

| File | Description |
| ---- | ----------- |
| vault/secrets.yml | Assorted keys used by the system encrypted via ansible-vault |
| siteconfig.yml | Used to tailor the vpc to a given site |
| servers.yml | Define what hosts to build and details about them |
| security_groups.yml | A list of firewall rules for use by the system |
| vpc.yml | A boilerplate file used to transform information in siteconfig.yml into something usable by Ansible |


## Created artifacts
After a full run creating a vpc, the system will create the following artifacts:

| Files | Description |
| ----- | ----------- |
| ${VPCNAME}-private.pem | The ssh key the hosts are configured with |
| vault/certs/* | A set of certificates specific to the vpc|
| kubernetes-manifests/* | A set of kubernetes manifests used to setup assorted services |


## Important Note
This git repo requires ansible 2.4 which changes the way ansible's internal encryption is managed.  This is not backwards compatible with ansible 2.3 and earlier versions.  If you run into issues with ansible not being able to decrypt content, it is likely due to using an older version of ansible.

Keep in mind, if you setup ansible 2.4 in your python virtualenv but the ansible playbooks in the git repo have not been updated to account for ansible 2.4, your playbooks will not work properly.  It is highly recommended that you maintain a different python virtual environment for ansbiel 2.3 vs ansible 2.4.


## Create a python virtual environment
```shell
virtualenv /path/to/local/pythonenv/ansible-2.4
source /path/to/local/pythonenv/ansible-2.4/bin/activate
pip install -U setuptools
pip install -r requirements.txt
```

** You might run into a problem on a linux machine where the python selinux libraries are not installed into your python vi
rtualenv.  If you see an error something like "Aborting, target uses selinux but python bindings (libselinux-python) aren't installed!", you will need to do a workaround.  The simplest workaround is to copy the system level python selinux library
 into your virtualenv.
```shell
cp -arp /usr/lib64/python2.7/site-packages/selinux /path/to/local/pythonenv/ansible-2.4/lib/python2.7/site-packages/
```

### Local Environment Setup
Cluster configurations are  under 'Secure Notes\Kubernetes Clusters'.  This path containes two files for each vpc to manage, a 'env' file which contains a set of variables to ex
port (and need to be tailored to your use case) and a 'pem' file used to ssh into the hosts.

### Export some variables needed by the system
```shell
export AWS_DEFAULT_REGION=us-east-1
export VPCNAME=k8s1
export AWS_ACCESS_KEY_ID=XXXXXXXXXXXXX
export AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXX
export SSH_USER_KEY=$HOME/keys/kubernetes/k8s1-private.pem
export KUBECONFIG=$HOME/.kube/k8s1-kubeconfig.yml
export ANSIBLE_VAULT_PASSWORD=XXXXXXXXXXXXXXXXX
```
*** Please see README.local-environment.md for more details

## Encrypted content
Generate encryption keys:

```shell
head -c 32 /dev/urandom | base64
```

Format of vars/${VPCNAME}/vault/secrets.yml:

```shell
auto_encrypt_password: "{{ lookup('env','ANSIBLE_VAULT_PASSWORD') }}"
kubeconfig_path: "{{ lookup('env', 'KUBECONFIG') }}"
cluster_token: XXXXXXXXXXXXXXXXXXXXXXX
weave_enc_key: XXXXXXXXXXXXXXXXXXXXXXX
secret_enc_key:
  key1: XXXXXXXXXXXXXXXXXXXXXXXXX

```

*** Please see the README.vault documentation for details on encryption

## Cluster Creation
### Prerequisites

- Route53 DNS zone is defined by 'k8s_domain_name' variable in vars/${VPCNAME}/siteconfig.yml.

- Exported AWS credentials (AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY) can manage:
	- VPCs
	- Subnets
	- Routing tables
	- Securoty groups
	- 'k8s_domain_name' Route53 DNS zone
	- EC2 instances and volumes
	- Keys
	- IMA Roles

- Parent DNS zone contains NS records for 'k8s_domain_name' sub-zone.


## Server Creation
Groups of servers are created via a cluster specific configuration at vars/{vpc_name}/servers.yml.  This lists each server to create and groups them by usage.


### Generate the certificates used by the system
```shell
ansible-playbook 00-certs.yml
```

### Create a VPC to contain the cluster
```shell
ansible-playbook 05-vpc.yml
```

### Build the servers defined in vars/${VPCNAME}/servers.yml
```shell
ansible-playbook 10-servers.yml
```

### Setup Python on CoreOS machines
```shell
ansible-playbook 20-coreos.yml
```

### Setup Kubernetes masters
```shell
ansible-playbook 30-cluster-masters.yml
```

### Setup Kubernetes workers
```shell
ansible-playbook 40-cluster-workers.yml
```

### Install extra addons into kubernetes
```shell
ansible-playbook 50-addons.yml
```

### Setup monitoring
```shell
ansible-playbook 60-monitoring.yml
```

## Server Management
To directly interact with a host's operating system, you can ssh into the system by passing through a bastion host.  Ansible is configured to handle pass through the bastion host automatically.

```shell
(local-python)[mbroome@localhost kubernetes-ansible]$ ./ssh-proxy.sh 172.18.97.5
54.207.28.169
Last login: Tue Jan  9 11:18:57 UTC 2018 from 172.18.96.5 on ssh
Container Linux by CoreOS stable (1576.4.0)
core@ip-172-18-97-5 ~ $ 
```

