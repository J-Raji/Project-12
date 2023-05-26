## Step 1 Jenkins Job Enhancement

-Webhook
http://54.207.140.176:8080/github-webhook/

-Branch
PRJ-12

--Jenkins page 52.67.64.253:8080 -admin password d84735f4243e4faf8ea684558518804e

1.Go to your Jenkins-Ansible server and create a new directory called ansible-config-artifact

`sudo mkdir /home/ubuntu/ansible-config-artifact`

2.Change permissions to this directory, so Jenkins could save files there 

`sudo chmod -R 777 /home/ubuntu/ansible-config-artifact`

3.Go to Jenkins web console -> Manage Jenkins -> Manage Plugins -> on Available tab search for Copy Artifact and install this plugin without restarting Jenkins

![copy artifact plugins installed](./Images/copy-artifact.jpg)

4.Create a new Freestyle project (you have done it in Project 9) and name it save_artifacts

![save artifact freestyle project created](./Images/save-artifact.jpgImages/save-artifact.jpg)

5.This project will be triggered by completion of your existing ansible project. Configure it accordingly:

![save artifact settings](./Images/save-artifact-setting1.jpg)

6.create a Build step and choose Copy artifacts from other project, specify ansible as a source project and /home/ubuntu/ansible-config-artifact as a target directory.

![save artifact settings2](./Images/save-artifact-setting2.jpg)

7.Test your set up by making some change in README.MD file inside your ansible-config-mgt repository (right inside master branch)

![build confirmed](./Images/build.jpg)

##Step 2 – Refactor Ansible code by importing other playbooks into site.yml

1.create a new file and name it site.yml

2.Create a new folder in root of the repository and name it static-assignments.

3.Move common.yml file into the newly created static-assignments folder

4.Inside site.yml file, import common.yml playbook

---
- hosts: all
- import_playbook: ../static-assignments/common.yml

![Structure for playbooks](./Images/playbooks.jpg)

5.Run ansible-playbook command against the dev environment

[]create another playbook under static-assignments and name it common-del.yml

---
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    yum:
      name: wireshark
      state: removed

- name: update LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
  - name: delete wireshark
    apt:
      name: wireshark-qt
      state: absent
      autoremove: yes
      purge: yes
      autoclean: yes

[]update site.yml with - import_playbook: ../static-assignments/common-del.yml instead of common.yml and run it against dev servers

`mkdir /home/ubuntu/ansible-config-mgt/` 

`cd /home/ubuntu/ansible-config-mgt/`

*BLOCKER FOUND*
Inventory and Playbooks not showing in ansible-config-artifact directory

`ansible all -m ping`

![localhost implicitly not listed](./Images/blocker.jpg)

** Resolution

[]`sudo apt remove ansible`

[]`sudo apt update`

[]`sudo apt install software-properties-common`

[]`sudo apt-add-repository --yes --update ppa:ansible/ansible`

[]`sudo apt install ansible`


`sudo ansible-playbook -i /inventory/dev.yml /playbooks/site.yml`

##Step 3 Configure UAT Webservers with a role ‘Webserver’

[]Launch two EC2 Instances
web1-uat
web2-uat

[] Create role

``
mkdir roles
cd roles
ansible-galaxy init webserver

``

[] update ansible-config-mgt/inventory/uat.yml

`sudo vi ansible-config-uat/inventory/uat.yml`

``
web2-uat 172.31.7.46
web2-uat 172.31.2.253

``

[]uncomment roles_path and provide full path to roles directory 

roles_path    = /home/ubuntu/ansible-config-mgt/roles

`sudo  vi /etc/ansible/ansible.cfg`

[]in main.yml add configuration file to do the following:

`cd ansible-config-mgt/roles/webserver/tasks`

`sudo vi main.yml`

`` 
---
- name: install apache
  become: true
  ansible.builtin.yum:
    name: "httpd"
    state: present

- name: install git
  become: true
  ansible.builtin.yum:
    name: "git"
    state: present

- name: clone a repo
  become: true
  ansible.builtin.git:
    repo: https://github.com/<your-name>/tooling.git
    dest: /var/www/html
    force: yes

- name: copy html content to one level up
  become: true
  command: cp -r /var/www/html/html/ /var/www/

- name: Start service httpd, if not started
  become: true
  ansible.builtin.service:
    name: httpd
    state: started

- name: recursively remove /var/www/html/html/ directory
  become: true
  ansible.builtin.file:
    path: /var/www/html/html
    state: absent
    
    ```

## step 4 Reference webserver roles

[]In static-assignments folder, create

`sudo vi static-assignments/uat-webservers.yml`

``
---
- hosts: uat-webservers
  roles:
     - webserver

`` 
[] Refer your uat-webservers.yml role inside site.yml

``
---
- hosts: all
- import_playbook: ../static-assignments/common.yml

- hosts: uat-webservers
- import_playbook: ../static-assignments/uat-webservers.yml
``

## step 5 commit & test

`sudo ansible-playbook -i inventory/uat.yml playbooks/site.yml`


[]
web2-uat http://172.31.7.46/index.php
web1-uat http://172.31.2.253/index.php

Webserver1=SHA256:C3YBErdvbJZPp60FVyoMyi3Brf6RroxHT3ddvxAZcws
Webserver20SHA256:CGRdfi8yus7eKK5JIIyac/eZ3tMudsxvJU2BNK1fPSw
NFS=SHA256:p1WqWdocA6TzqtcFpvIVllb7B1zwsb0Kwmv4H2UhGKo
Nginx LB=SHA256:bSnnStPd8mZ5Aa4CUNuF0Jx8wC0Iz9nzj2EnODCuho8
Db=SHA256:Hz8bhs3TG1gTxhnijszifQ+4aGOsJH77NrA4jE6hKos

## Step3

[]