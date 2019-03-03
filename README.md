1) Create two EC2 Instances in AWS Cloud.
a) Using Ansible
pre-requisites:
i) Ansible must me installed (dependencies: python, python-devel, python-pip, openssl)
ii) awscli & boto must be installed (pip install awscli, pip install boto)
iii) aws configure must be done (access_key, secret_key, region)
iv) ec2.py & ec2.ini must be installed

#create a demo.yml file with below content
---
- name: Provision instances in AWS
  hosts: localhost
  connection: local
  gather_facts: False
  # loading AWS variables from below file
  vars_files:
  - group_vars/all

  tasks:
  - name: MSR-test-instance-1
    ec2:
      access_key: "{{ ec2_access_key }}"
      secret_key: "{{ ec2_secret_key }}"
      keypair: "{{ ec2_keypair }}"
      group: "{{ ec2_security_group }}"
      type: "{{ ec2_instance_type }}"
      image: "{{ ec2_image }}"
      region: "{{ ec2_region }}"
      instance_tags: "{'Name':'MSR-test-instance-1'}"
      wait: true

  - name: MSR-test-instance-2
    ec2:
      access_key: "{{ ec2_access_key }}"
      secret_key: "{{ ec2_secret_key }}"
      keypair: "{{ ec2_keypair }}"
      group: "{{ ec2_security_group }}"
      type: "{{ ec2_instance_type }}"
      image: "{{ ec2_image }}"
      region: "{{ ec2_region }}"
      instance_tags: "{'Name':'MSR-test-instance-2'}"
      wait: true

#group_vars/all
---
# AWS specific variables
ec2_access_key: XXXXXXXXXXXXXXXX
ec2_secret_key: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
ec2_region: ap-south-1
ec2_image: ami-04ea996e7a3e7ad6b
ec2_instance_type: t2.micro
ec2_keypair: mumbaikeypair
ec2_security_group: default
ec2_tag_name_prefix: test
ec2_hosts: all
wait_for_port: 22

#execute .yml file as below to create two instances

$ ansible-playbook demo.yml

########################################################################################################################################

b) using Terraform
pre-requisites:
i) terraform must be installed

#create a file msrtest.tf with below content
provider “aws” {
	region = “ap-south-1”
}
resource “aws_instance” “MSR-test-Instance-1” {
	ami = ami-04ea996e7a3e7ad6b
	instance_type = “t2.micro”
	key_name = "specify key pair"
	tags = {
		Name = “MSR-test-Instance-1”
	}
}

resource “aws_instance” “MSR-test-Instance-2” {
	ami = ami-04ea996e7a3e7ad6b
	instance_type = “t2.micro”
	key_name = "specify key pair"
	tags = {
		Name = “MSR-test-Instance-2”
	}
}

#execute the terraform script to create two instances as below

$ teraform init msrtest.tf
$ teraform plan msrtest.tf
$ teraform apply msrtest.tf 

######################################################################################################################################

2) Once these two servers are provisioned, ensure the below following software packages are installed using configuration management tool in both the provisioned instances.

Using Ansible:
#create a msrpackages.yml file with below content
---
- hosts: all
  remote_user: ubuntu
  gather_facts: yes
  become: yes
  become_user: root
  become_method: sudo

  tasks:
  - name: installing required packages
    apt:
     name: ['software-properties-common', 'openssl', 'git']
     state: present

  - name: getting pip
    shell: curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"

  - name: installing pip
    shell: python3 get-pip.py

  - name: Install nvm-0.33.2
    shell: >
      curl https://raw.githubusercontent.com/creationix/nvm/v0.33.2/install.sh | sh
      creates=/home/{{ ansible_user_id }}/.nvm/nvm.sh

  - name: Installing node and set version-8.12.0
    shell: >
      /bin/bash -c "source ~/.nvm/nvm.sh && nvm install 8.12.0 && nvm alias default 8.12.0"
      creates=/home/{{ ansible_user_id }}/.nvm/alia

  - name: adding docker gpg
    shell: curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

  - name: adding docker repo
    shell: add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable"

  - name: updating all packages
    shell: apt update

  - name: installing docker-engine latest
    package:
      name: docker-ce
      state: present
    notify:
       - startservice

  - name: installing docker-compose latest
    pip:
      name: docker-compose
      state: latest

  handlers:
  - name: startservice
    service:
        name: docker
        state: restarted

#for achiving desired results of installing packages on remote hosts execute below command.

$ ansible-playbook -i ec2.py msrpackages.yml -e ansible_python_interpreter=/usr/bin/python3 

########################################################################################################################################

3) Create a Docker Container in MSR-test-Instance-1 using Docker Compose file and ensure apache webserver is installed. Try to use configuration management tools to automate the entire installation of apache and deploy a sample html file from a GitHub repository.

Using Ansible: to create docker container of Apache webserver on  MSR-test-Instance-1.

1) first create a docker-compose.yml file in /tmp on Ansible master with below details 
version: '3'
services:
  apache:
    image: bitnami/apache
    container_name: webapp
    ports:
      - 80:8080
    volumes:
      - /tmp/html:/opt/bitnami/apache/htdocs

2) now create an docker-webapp.yml file to achive above requirement.
---
- hosts: 13.233.207.188
  remote_user: ubuntu
  become: yes
  gather_facts: yes
  become_user: root
  become_method: sudo

  tasks:
  - name: Upload a file to the remote host
    copy:
      src: /tmp/docker-compose.yml
      dest: /var/lib/docker/
      mode: 0755

  - name: pulling index.html from git hub
    shell: >
         cd /tmp && git clone https://github.com/siripurapubharath/html.git

  - name: move to /var/lib/docker & bringing up container
    shell: >
         cd /var/lib/docker && docker-compose up -d

#now to achive the above requirement execute the below command.

$ ansible-playbook -i ec2.py docker-webapp.yml -e ansible_python_interpreter=/usr/bin/python3

########################################################################################################################################

4) Create a Docker Container in MSR-test-Instance-2 using Docker Compose file and ensure CouchDB  Database is installed. Try to use any configuration management tool to automate the entire installation processes. 

Using Ansible: to create docker container of CouchDB on  MSR-test-Instance-2.

1) first create a docker-compose.yml file in /tmp on Ansible master with below details 
version: '3'
services:
   db:
      image: couchdb:latest
      container_name: couch_db
      ports:
         - 80:5984

2) now create an docker-db.yml file to achive above requirement.
---
- hosts: 13.233.207.188
  remote_user: ubuntu
  become: yes
  gather_facts: yes
  become_user: root
  become_method: sudo

  tasks:
  - name: Upload a file to the remote host
    copy:
      src: /tmp/docker-compose.yml
      dest: /var/lib/docker/
      mode: 0755

  - name: move to /var/lib/docker & bringing up container
    shell: >
   cd /var/lib/docker && docker-compose up -d

#now to achive the above requirement execute the below command.

$ ansible-playbook  -i ec2.py docker-db.yml -e ansible_python_interpreter=/usr/bin/python3
