# Ansible Realtime Project

In this Ansible realtime project, we create three EC2 instances on AWS.
Out of the three instances, two EC2 instances use the Ubuntu distribution and one EC2 instance uses the Amazon Linux distribution.

The EC2 instances are created using the localhost connection.
We also set up passwordless authentication between the Ansible control node and the newly created EC2 instances.
Finally, we automate the shutdown of only Ubuntu instances using Ansible conditionals.

# Task 1: Creating EC2 instances
1. Create three EC2 instances on AWS using Ansible loops:
- Two instances with Ubuntu distribution
- One instance with Amazon Linux distribution

2. Secure the secrets and passwords using Ansible Vault (vault.pass).
- AWS Access Key and Secret Access Key are stored in pass.yaml to keep credentials secure.

### Installing the ansible galaxy collection of amazon
 ```
 ansible-galaxy collection install amazon.aws
 ```

 ### Creating a password for the vault
```
openssl rand -base64 2048 > vault.pass
```
### Add your AWS credentials using Ansible Vault
```
ansible-vault create group_vars/all/pass.yaml --vault-password-file vault.pass
```

This file securely stores the AWS Access Key and Secret Access Key.


### ec2_create.yaml
```
---
- hosts: localhost
  connection: local

  tasks: 
  - name: Launch EC2 Instance
    amazon.aws.ec2_instance:
      name: "{{ item.name }}"
      key_name: "aws-key"
      instance_type: t2.micro
      security_group: default
      region: "{{ aws_region }}"
      aws_access_key: "{{ ec2_access_key }}" #From another variable file (ansible vault)
      aws_secret_key: "{{ ec2_secret_key }}" #From another variable file (ansible vault)
      network:
        assign_public_ip: true
      image_id: "{{ item.image_id }}"
      tags:
        Name: "{{ item.name }}" 
    loop:
      - { image_id: "{{ ami_ubuntu }}", name: "manage-node-1", os: "Ubuntu" }
      - { image_id: "{{ ami_ubuntu }}", name: "manage-node-2", os: "Ubuntu" }
      - { image_id: "{{ ami_amazon }}", name: "manage-node-3", os: "Amazon Linux" }
    tags:
        - ec2_create

```
 This playbook launches multiple EC2 instances using a loop, allowing different AMIs and instance names to be defined in a single task. It simplifies EC2 provisioning and reduces code duplication by reusing the same configuration for Ubuntu and Amazon Linux instances.


### Running the Ansible playbook to automate tasks
```
ansible-playbook ec2_create.yaml --vault-password-file vault.pass
```
This command creates the EC2 instances using the encrypted AWS credentials.

# Task 2: Setting up the Passwordless Authentication for the created EC2 instance
To set up passwordless authentication between the Ansible control node and the EC2 instances, run the following command:
```
ssh-copy-id -f "-o IdentityFile <PATH TO PEM FILE>" ubuntu@<INSTANCE-PUBLIC-IP>
```

- ssh-copy-id: This is the command used to copy your public key to a remote machine.
- -f: This flag forces the copying of keys, which can be useful if you have keys already set up and want to overwrite them.
- "-o IdentityFile <PATH TO PEM FILE>": This option specifies the identity file (private key) to use for the connection. The -o flag passes this option to the underlying ssh command.
- ubuntu@<INSTANCE-IP>: This is the username (ubuntu) and the IP address of the remote server you want to access.


# Task 3: Automate the shutdown of only Ubuntu Instances only using Ansible COnditionals
We use Ansible conditionals to shut down only Ubuntu instances. 

### ec2stop.yaml
```
---
- hosts: all
  become: true
  tasks:
  - name: Shutdown Ubuntu instances only
    ansible.builtin.shell: shutdown -h now
    when: ansible_distribution == "Ubuntu"
    tags:
      - shutdown
    
```
 This playbook shuts down only Ubuntu-based EC2 instances by using a conditional check. The task runs the shutdown command exclusively when the target host's operating system is detected as Ubuntu, ensuring other Linux distributions are not affected.


### Stops and shutdown the EC2 instances
```
ansible-playbook -i inventory.ini ./ec2_stop.yaml
```
This playbook shuts down only the Ubuntu EC2 instances, while the Amazon Linux instance remains running.

## Project Directory Structure
```
├── ec2_create.yaml
├── ec2_stop.yaml
├── group_vars
│   └── all
│       └── pass.yaml
├── inventory.ini
└── vault.pass
```