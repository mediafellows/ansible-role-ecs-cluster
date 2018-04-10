[![Build Status](https://travis-ci.org/mediapeers/ansible-role-ecs-cluster.svg?branch=master)](https://travis-ci.org/mediapeers/ansible-role-ecs-cluster)

# ansible-role-ecs-cluster

Ansible role for simplifying the provisioning and decommissioning of Auto-scaling ECS clusters within an AWS account.

For more detailed on information on the creating:

* Auto-scaling Groups: http://docs.ansible.com/ansible/ec2_asg_module.html;
* ECS Clusters: http://docs.ansible.com/ansible/ecs_cluster_module.html;
* ECS Services: http://docs.ansible.com/ansible/ecs_service_module.html;
* ECS Task Definition: http://docs.ansible.com/ansible/ecs_taskdefinition_module.html.

This role will completely setup an unlimited size, self-healing, auto-scaling EC2 cluster registered to an ECS cluster, ready to accept ECS Service and Task Definitions with centralised log management.

## Requirements

Requires the latest Ansible EC2 support modules along with Boto.

You will also need to configure your Ansible environment for use with AWS, see http://docs.ansible.com/ansible/guide_aws.html.

## Role Variables

Required variables:

* `ecs_cluster_name` - You must specify the name of the ECS cluster, e.g. my-cluster
* `ecs_ssh_key_name` - You must specify the name of the SSH key you want to assign to the EC2 instances, e.g. my-ssh-key
* `ecs_security_groups` - You must specify a list of existing EC2 security groups IDs to apply to the auto-scaling EC2 instances, e.g. ['sg-1234']
* `ecs_vpc_subnets` - You must specify a list of existing VPC subnet ids for which to provision the EC2 nodes into, e.g. ['subnet-123', 'subnet-456']

For overwriting other variables (with defaults) checkout `defaults/main.yml` for reference.

The default `ecs_userdata` will register the EC2 instance within the ECS cluster and configure the instance to stream it's logs to AWS CloudWatch Logs for centralised management.  Log Groups are pre-pended with `{{ application_name }}-{{ env }}`

## Dependencies

Depends on no other Ansible roles.

## Example Playbook

Before using this role you will need to install the role, the simplist way to do this is: `ansible-galaxy install mediapeers.ansible-role-ecs-cluster`.

For completeness the examples creates

* VPC to hold the ECS cluster, using the role: `daniel-rhoades.aws-vpc`;
* EC2 Security Groups to apply to the EC2 instances, using the role: `daniel-rhoades.aws-security-group`.

```
- name: My System | Provision all required infrastructure
  hosts: localhost
  connection: local
  gather_facts: no
  vars:
    my_vpc_name: "my_example_vpc"
    my_vpc_region: "eu-west-1"
    my_vpc_cidr: "172.40.0.0/16"
    everywhere_cidr: "0.0.0.0/0"

    # Subnets within the VPC
    my_vpc_subnets:
      - cidr: "172.40.10.0/24"
        az: "{{ my_vpc_region }}a"

      - cidr: "172.40.20.0/24"
        az: "{{ my_vpc_region }}b"

    # Allow the subnets to route to the outside world
    my_public_subnet_routes:
      - subnets:
          - "{{ my_vpc_subnets[0].cidr }}"
          - "{{ my_vpc_subnets[1].cidr }}"
        routes:
          - dest: "{{ everywhere_cidr }}"
            gw: igw

    # Inbound security groups, e.g. public facing services like a load balancer
    my_inbound_security_groups:
      - sg_name: inbound-web
        sg_description: allow http and https access (public)
        sg_rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: "{{ everywhere_cidr }}"

          - proto: tcp
            from_port: 443
            to_port: 443
            cidr_ip: "{{ everywhere_cidr }}"

      # Only allow SSH access from within the VPC, to access any services within the VPC you would need to create a
      # temporary bastion host
      - sg_name: inbound-ssh
        sg_description: allow ssh access
        sg_rules:
         - proto: tcp
           from_port: 22
           to_port: 22
           cidr_ip: "{{ my_vpc_cidr }}"

    # Internal inbound security groups, e.g. services which should not be directly accessed outside the VPC, such as
    # the web servers behind the load balancer.
    #
    # This has to be a file as it needs to be dynamically included after the inbound security groups have been created
    my_internal_inbound_security_groups_file: "internal-securitygroups.yml"

    # Outbound rules, e.g. what services can the web servers access by themselves
    my_outbound_security_groups:
      - sg_name: outbound-all
        sg_description: allows outbound traffic to any IP address
        sg_rules:
          - proto: all
            cidr_ip: "{{ everywhere_cidr }}"

    # Name of the SSH key registered in IAM
    my_ec2_key_name: "{{ ssh_key_name }}"

    # Name of the ECS cluster to create
    my_ecs_cluster_name: "my-cluster"

  roles:
    # Provision networking
    - {
        role: daniel-rhoades.aws-vpc,
        vpc_name: "{{ my_vpc_name }}",
        vpc_region: "{{ my_vpc_region }}",
        vpc_cidr_block: "{{ my_vpc_cidr }}",
        vpc_subnets: "{{ my_vpc_subnets }}",
        public_subnet_routes: "{{ my_public_subnet_routes }}"
      }

    # Provision security groups
    - {
        role: daniel-rhoades.aws-security-groups,
        vpc_region: "{{ my_vpc_region }}",
        vpc_id: "{{ vpc.vpc_id }}",
        ec2_group_inbound_sg: "{{ my_inbound_security_groups }}",
        ec2_group_internal_inbound_sg_file: "{{ my_internal_inbound_security_groups_file }}",
        ec2_group_outbound_sg: "{{ my_outbound_security_groups }}"
      }

    # Provision ECS with auto scaling EC2 instances
    - {
        role: daniel-rhoades.aws-ecs-autoscale,
        ecs_cluster_name: "{{ my_ecs_cluster_name }}",
        ec2_security_groups: [
          "{{ ec2_group_internal_inbound_sg.results[0].group_id }}",
          "{{ ec2_group_outbound_sg.results[0].group_id }}"
          ],
        ec2_asg_availability_zones: [
          "{{ my_vpc_subnets[0].az }}",
          "{{ my_vpc_subnets[1].az }}"
          ],
        ec2_asg_vpc_subnets: [
          "{{ vpc.subnets[0].id }}",
          "{{ vpc.subnets[1].id }}"
          ],
        vpc_name: "{{ my_vpc_name }}",
        key_name: "{{ my_ec2_key_name }}"
      }
```

The example `internal-securitygroups.yml` looks like:

```
ec2_group_internal_inbound_sg:
  - sg_name: inbound-web-internal
    sg_description: allow http and https access (from load balancer only)
    sg_rules:
      - proto: tcp
        from_port: 80
        to_port: 80
        group_id: "{{ ec2_group_inbound_sg.results[0].group_id }}"
```

To decommission the groups:

```
- name: My System | Decommission all required infrastructure
  hosts: localhost
  connection: local
  gather_facts: no
  vars:
    my_vpc_name: "my_example_vpc"
    my_vpc_region: "eu-west-1"
    my_vpc_cidr: "172.40.0.0/16"
    everywhere_cidr: "0.0.0.0/0"

    # Subnets within the VPC
    my_vpc_subnets:
      - cidr: "172.40.10.0/24"
        az: "{{ my_vpc_region }}a"

      - cidr: "172.40.20.0/24"
        az: "{{ my_vpc_region }}b"

    # Allow the subnets to route to the outside world
    my_public_subnet_routes:
      - subnets:
          - "{{ my_vpc_subnets[0].cidr }}"
          - "{{ my_vpc_subnets[1].cidr }}"
        routes:
          - dest: "{{ everywhere_cidr }}"
            gw: igw

    # Name of the ECS cluster to create
    my_ecs_cluster_name: "my-cluster"

  roles:
    # Provision networking
    - {
        role: daniel-rhoades.aws-vpc,
        vpc_name: "{{ my_vpc_name }}",
        vpc_region: "{{ my_vpc_region }}",
        vpc_cidr_block: "{{ my_vpc_cidr }}",
        vpc_subnets: "{{ my_vpc_subnets }}",
        public_subnet_routes: "{{ my_public_subnet_routes }}"
      }

    # Decommision ECS with auto scaling EC2 instances
    - {
        role: daniel-rhoades.aws-ecs-autoscale,
        ecs_cluster_name: "{{ my_ecs_cluster_name }}",
        vpc_name: "{{ my_vpc_name }}",
        ecs_state: "absent"
      }
```

## License

MIT

## Author Information

Daniel Rhoades (https://github.com/daniel-rhoades)
