# Spinup EC2 and create dynamic invetory : Ansible 

Here we am creating an EC2 instance and going to add a test page to created EC2 via a single playbook. 

Since the instance IP can be opbtained only after creation, we can't use static inventory file. Hence we need to create a dynamic inventory file.
Dynamic invetory is a feature on ansible that it will store the host file details internally. 

I used add_host module to create dynamic inventory. 

Here is my playbook

```sh
---
- name: "Creating AWS instance using Ansible"
  hosts: localhost
  vars_files:
  - access_key.vars
  vars:
    key_name: "ansible-key"
  tasks:
  - name: "creating key pair"
    ec2_key:
      access_key: "{{access_key}}"
      secret_key: "{{secret_key}}"
      region: "{{ region }}"
      name: "{{ key_name }}"
      state:  present
    register: keypair
  - name: "copying Private Key"
    when: keypair.changed == true
    copy:
      content: "{{keypair.key.private_key}}"
      dest: "{{ key_name}}.pem"
      mode: 0400
  - name: "creating security group"
    ec2_group:
      access_key: "{{access_key}}"
      secret_key: "{{secret_key}}"
      name: ansible-sg
      tags: 
        Name: "SG"
      description: sg using ansible
      region: "{{region}}" 
      rules:
      - proto: tcp
        from_port: 80
        to_port: 80
        cidr_ip: 0.0.0.0/0
      - proto: tcp
        from_port: 22
        to_port: 22
        cidr_ip: 0.0.0.0/0
    register: sggroup
  - name: " Creatng webserver"
    ec2:
      access_key: "{{access_key}}"
      secret_key: "{{secret_key}}"
      region: "{{ region }}"
      key_name: "{{ key_name }}"
      group_id:
      - "{{sggroup.group_id}}"
      instance_tags:
        Name: "{{ instance_tag }}"
      instance_type: t2.micro
      image: "ami-010aff33ed5991201"
      count_tag:
        Name: "{{ instance_tag }}"
      exact_count: 1 
    register: ec2_info
  - name: " Pausing the playbook to make the EC2 up"
    pause:
      seconds: 40

  - name: "Aws - Creating Dynamic Invetory Of Ec2 Instances"
    add_host:
      hostname: "{{ item.private_ip }}"
      groupname: webserver
      ansible_ssh_private_key_file: "{{ key_name}}.pem"
      ansible_user: "ec2-user"
      ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
    with_items:
    - "{{ec2_info.tagged_instances }}"
  - pause:
      seconds: 10
- name: host
  hosts: webserver
  become: true
  tasks:
  - name: Installing apache 
    yum:
      name: httpd
      state: present
    register: apache_status
  - name: Restart HTTPD
    when: apache_status.changed == true
    service:
      name: httpd
      state: restarted
  - name: creating index page
    copy:
      content: "<center><h1> This is Sample Index.html </h1></center>"
      dest: /var/www/html/index.html
      owner: apache
      group: apache
```
> 
#### Count tag - Exact count
```sh
count_tag:
        Name: "{{ instance_tag }}"
      exact_count: 1 
```
Here I used count tag - exact count, which means the instance with tag name mentioned, should be created on same nos as mentioned in exact_count 
> 
#### Add host
```sh
    add_host:
      hostname: "{{ item.private_ip }}"
      groupname: webserver
      ansible_ssh_private_key_file: "{{ key_name}}.pem"
      ansible_user: "ec2-user"
      ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
    with_items:
    - "{{ec2_info.tagged_instances }}"
```
This will create a dynamic inventory and store internally so that once the instance is provisioned, it's details will get added to the internal dynamic inventory file. 












------------------------------------------ _Gigin George_ ---------------------------------------------------------------

--------------------------------------- _linkedin.com/in/gigin-george-648679142_ -------------------------------------

---------------------------------- _gigingkallumkal@gmail.com_ ---------------------------------------------------------
