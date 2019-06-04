# AWS services backup using Ansible playbooks
Ansible playbook that takes AWS [RDS](https://aws.amazon.com/rds/), [EFS](https://aws.amazon.com/efs/) (using [AWS Backup](https://aws.amazon.com/backup/)), [EC2](https://aws.amazon.com/ec2/) snapshots and application(ex. [Neo4j](https://neo4j.com/) DB on Ubuntu EC2 instance) backup in parallel

This is a sample Ansible playbook solution that executes below flow :

![Alt text](https://raw.githubusercontent.com/sanket-bengali/aws-backup-ansible/master/images/Ansible%20Playbook%20flow.png?raw=True "Ansible playbook flow")
## Assumptions
1. Dependencies :
For this solution, it is assumed that the [Bastian host](https://en.wikipedia.org/wiki/Bastion_host) has necessary dependencies installed like [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html), [Boto](https://aws.amazon.com/sdk-for-python/) (to execute AWS operations), etc.
More info on [AWS Linux Bastian host](https://docs.aws.amazon.com/quickstart/latest/linux-bastion/architecture.html)

2. Ansible operations from playbooks :
   [Ansible supported AWS modules](https://docs.ansible.com/ansible/latest/modules/list_of_cloud_modules.html#amazon)

   For other AWS operations, AWS CLI is used. [Configuring the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)

3. SSH key of the <my_app> EC2 instance is copied from S3 bucket into Bastian host to perform SSH operations.

4. AWS constants :
Some variables for AWS operations like region, account_id are stored as constants in vars files. This could to be enhanced to be dynamic.

5. Pre and post scripts :
As pre-script, httpd service is stopped and as post-script, httpd service is started. This could be enhanced to run any scripts.

6. This sample solution doesn't include automated/scheduled backup option. It can be enhanced as needed.

## Playbooks structure
![Alt text](https://raw.githubusercontent.com/sanket-bengali/aws-backup-ansible/master/images/Ansible%20playbook%20tree%20structure.png?raw=True "Ansible playboks tree structure")

## Inventories
The inventories section contains the hosts information [Bastian host + <my_app> EC2 instances].

It also includes common variables (like AWS region, account_id etc.) across all hosts and roles that are used for executing AWS operations.

#### NOTE : Values in these files need to be updated accordingly.

## Roles
The playbook is divided into roles as shown in above tree structure.

Each role has its own vars (can be changed as needed) and tasks directories mentioning the operations to be performed by that role.

## Running the playbook

#### NOTE : Running these playbooks uses AWS services and creates Backup resources, which could add cost as per AWS pricing.

1. Clone this repository : ```git clone https://github.com/sanket-bengali/aws-backup-ansible.git```

2. Go to the playbook directory : ```cd /path/to/repository/ansible/aws/```

3. Update inventory files variables

   In "inventories/poc/hosts" : <ec2_public_ip>, <ec2_user>, <neo4j_ec2_public_ip>

   In "inventories/poc/group_vars/all.yaml" : "aws-region", "aws-account-id"

4. Update playbooks variables inside "roles"
   
   #### NOTE : [Passing playbook variables on the command line](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#passing-variables-on-the-command-line)

   a. get_ssh_key
   
   -> In "get_ssh_key/tasks/main.yaml" : <bucket_name>, <path_to_ssh_key/key_name.pem>
   
   -> In "get_ssh_key/vars/main.yaml" : <key_name.pem>
   
   b. take_ec2_snapshot
   
   -> In "take_ec2_snapshot/vars/main.yaml" : "my-app-ec2-instance-name", "ec2_device_name"
   
   c. take_efs_backup
   
   -> In "take_efs_backup/vars/main.yaml" : "my-app-efs-name", "efs-backup-vault-name"
   
   d. take_rds_snapshot
   
   -> In "take_rds_snapshot/vars/main.yaml" : "my-app-pgsql-db"
   
   e. take_neo4j_db_backup
   
   -> In "take_neo4j_db_backup/vars/main.yaml" : "/home/ubuntu/<neo4j_backup_dir>"

5. Run the playbook : ```ansible-playbook my_app_backup.yaml -i inventories/poc/hosts```

## More information

[AWS services backup using Ansible playbooks](https://medium.com/@sanketbengali.23/aws-services-backup-using-ansible-playbooks-df8516a0e2b5)

## License

The MIT License (MIT). Please see [License File](LICENSE) for more information.
