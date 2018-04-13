[![Build Status](https://travis-ci.org/mediapeers/ansible-role-ecs-cluster.svg?branch=master)](https://travis-ci.org/mediapeers/ansible-role-ecs-cluster)

# Ansible role for creating an AWS ECS-Cluster

Ansible role for simplifying the provisioning and decommissioning of Auto-scaling ECS clusters within an AWS account.

For more detailed on information on the creating:

* Auto-scaling Groups: http://docs.ansible.com/ansible/ec2_asg_module.html;
* ECS Clusters: http://docs.ansible.com/ansible/ecs_cluster_module.html;
* ECS Services: http://docs.ansible.com/ansible/ecs_service_module.html;
* ECS Task Definition: http://docs.ansible.com/ansible/ecs_taskdefinition_module.html.

This role will completely setup an unlimited size, self-healing, auto-scaling EC2 cluster registered to an ECS cluster, ready to accept ECS Service and Task Definitions with centralised log management.

## Installation

Install this role through your usual patterns into your Ansible project. Either use `ansible-galaxy install mediapeers.ecs-cluster` or add it to your roles
dir by adding it as a git submodule.

## Requirements

Needs a VPC with subnets and security groups to be present on AWS already for this role to use. Also you should use Ansible 2.5 or newer.

If not already done you also need to configure your Ansible environment for use with AWS, see http://docs.ansible.com/ansible/guide_aws.html.

## Role Variables

Required variables:

* `ecs_cluster_name` - You must specify the name of the ECS cluster, e.g. my-cluster
* `ecs_ssh_key_name` - You must specify the name of the SSH key you want to assign to the EC2 instances, e.g. my-ssh-key
* `ecs_security_groups` - You must specify a list of existing EC2 security groups IDs to apply to the auto-scaling EC2 instances, e.g. ['sg-1234']
* `ecs_vpc_subnets` - You must specify a list of existing VPC subnet ids for which to provision the EC2 nodes into, e.g. ['subnet-123', 'subnet-456']

For overwriting other variables (with defaults) checkout `defaults/main.yml` for reference.

**Notes:**

Make sure that `ecs_ec2_region` is set correctly. Machine tagging can be done by setting `ecs_ec2_tags`.

The default `ecs_userdata` will register the EC2 instance within the ECS cluster and configure the instance to stream it's logs to AWS CloudWatch Logs for centralised management.
Log Groups are pre-pended with `{{ cloudwatch_namespace }}`

## Dependencies

Depends on no other Ansible roles.

## Example Playbook

After installing this role you can use it in your project as `mediapeers.ecs-cluster`.

For completness example contains two plays. One to fullfil preconditions for this role by setting up a VPC with normal Ansible modules. And a second
one using this role to setup the ECS cluster using the results of the VPC setup.


```
- name: Setup example networking (VPC, Security-Groups and Subnets)
  hosts: localhost
  tasks:

    - name: Create VPC
      ec2_vpc_net:
        name: 'My test VPC'
        cidr_block: 10.10.0.0/16
        region: us-east-1
        state: present
      register: my_vpc

    - name: Create Subnet 1 (in AZ A)
      ec2_vpc_subnet:
        vpc_id: "{{ my_vpc.vpc.id }}"
        cidr: 10.10.1.0/24
        az: us-east-1a
        region: us-east-1
        state: present
      register: my_subnet_1

    - name: Create Subnet 2 (in AZ B)
      ec2_vpc_subnet:
        vpc_id: "{{ my_vpc.vpc.id }}"
        cidr: 10.10.2.0/24
        az: us-east-1b
        region: us-east-1
        state: present
      register: my_subnet_2

    - name: Create Security group for Web traffic
      ec2_group:
        name: ECS-Webtraffic
        vpc_id: "{{ my_vpc.vpc.id }}"
        region: us-east-1
        rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0
        state: present
      register: my_sec_group

    - name: Create SSH key
      ec2_key:
        name: 'my_ssh_key'
        wait: yes
        state: present
        region: us-east-1

- name: Setup EC2 cluster inside the VPC
  hosts: localhost
  vars:
    ecs_cluster_name: 'my-new-ecs-cluster'
    ecs_ssh_key_name: 'my_ssh_key'
    ecs_security_groups:
      - "{{ my_sec_group.group_id }}"
    ecs_vpc_subnets:
      - "{{ my_subnet_1.id }}"
      - "{{ my_subnet_2.id }}"
    ecs_asg_min_size: 2
    ecs_asg_max_size: 4
    ecs_asg_desired_capacity: 2
    ecs_ec2_tags:
      - Name: "my-ec2-cluster-instance"
      - role: "ecs-cluster"
    ecs_ec2_region: us-east-1
  roles:
    - mediapeers.ecs-cluster

```

## License

MIT

## Author Information

- Stefan Horning
- Daniel Rhoades (https://github.com/daniel-rhoades)
