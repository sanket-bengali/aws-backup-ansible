## The main playbook "my_app_backup.yaml" executes other playbooks (roles) in below sequence :

1. Copy SSH key from S3 bucket to Bastian host

2. Run pre-script

3. Execute 4 backups in parallel

4. Run post-script

```
---
- hosts: localhost
  gather_facts: False
  roles:
    - role: get_ssh_key

- hosts: my-app-ec2-host
  gather_facts: False
  roles:
    - role: run_pre_script

- hosts: my_app:bastian
  gather_facts: False
  strategy: free
  tasks:
    - include_role:
        name: take_rds_snapshot
      when: "inventory_hostname == 'localhost_rds_backup' in groups['bastian']"
    - include_role:
        name: take_efs_backup
      when: "inventory_hostname == 'localhost_efs_backup' in groups['bastian']"
    - include_role: 
        name: take_ec2_snapshot
      when: "inventory_hostname == 'localhost_ec2_backup' in groups['bastian']"
    - include_role: 
        name: take_neo4j_db_backup
      when: "inventory_hostname == 'neo4j-ec2-host' in groups['my_app']"

- hosts: my-app-ec2-host
  gather_facts: False
  roles:
    - role: run_post_script
```

### To copy SSH key from S3 to Bastian host, Ansible AWS module is used :

```
- name: Copy SSH key to Bastian EC2 instance
  aws_s3:
    bucket: <bucket_name>
    object: <path_to_ssh_key/key_name.pem>
    dest: "{{ ssh_key_local_path }}"
    mode: get

- name: Change key mode to 600
  file:
    path: "{{ ssh_key_local_path }}"
    mode: 0600
```

### To execute pre and post scripts, Ansible module is used :

```
- name: Run pre-script on EC2 instance
  service: name=httpd state=stopped
  become: yes
  become_method: sudo
  register: pre_script_output
```

### For EC2 snapshot, Ansible AWS module is used :

```
- name: Create Snapshot of volume mounted on device_name attached to instance_id
  ec2_snapshot:
    region: "{{ region }}"
    instance_id: "{{ instance_id }}"
    device_name: "{{ device_name }}"
```

### For RDS snapshot, Ansible AWS module is used :

```
- name: Create Postgres DB snapshot and wait for it to become available
  rds:
    command: snapshot
    region: "{{ region }}"
    instance_name: "{{ pgsql_db_instance_name }}"
    snapshot: "{{ pgsql_db_instance_name }}-{{ current_time }}"
    wait: yes
    wait_timeout: 600
```

### For EFS backup, there is no in-built Ansible AWS module available. Hence, AWS CLI is executed :

```
- name: Start EFS backup job
  command: >
    aws backup start-backup-job --backup-vault-name {{ backup_vault_name }} --resource-arn arn:aws:elasticfilesystem:{{ region }}:{{ account_id }}:file-system/{{ file_system_id }} --iam-role-arn arn:aws:iam::{{ account_id }}:role/service-role/{{ backup_service_role }}
  changed_when: false
  register: aws_efs_backup_job

- name: Print EFS Backup job id
  debug:
    msg: "EFS backup job id : {{ (aws_efs_backup_job.stdout|from_json)['BackupJobId'] }}"

- name: Store EFS Backup job id in fact
  set_fact:
    efs_backup_job_id: "{{ (aws_efs_backup_job.stdout|from_json)['BackupJobId'] }}"

- name: Wait until the EFS Backup job is completed
  command: >
    aws backup describe-backup-job --backup-job-id {{ efs_backup_job_id }}
  register: efs_backup_job_details
  until: (efs_backup_job_details.stdout|from_json)['State'] == "COMPLETED" or (efs_backup_job_details.stdout|from_json)['State'] == "FAILED" or (efs_backup_job_details.stdout|from_json)['State'] == "ABORTED" or (efs_backup_job_details.stdout|from_json)['State'] == "EXPIRED"
  delay: 10
  retries: 10
```

### For Neo4j DB backup, [Neo4j's CLI](https://neo4j.com/docs/operations-manual/current/backup/performing/) is executed to take the DB backup :

```
- name: Create Neo4j DB backup
  command: >
    neo4j-admin backup --backup-dir={{ backup_dir }} --name={{ backup_name }}
  changed_when: true
  become: yes
  become_method: sudo
  register: neo4j_db_backup_output

- name: Print Neo4j DB backup output
  debug:
    msg: "Neo4j DB backup output : {{ neo4j_db_backup_output.stdout }}"
```
