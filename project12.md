## ANSIBLE_REFACTORING_ASSIGNMENTS

## Jenkins CI/CD on a 3-tier application && Ansible Configuration Management Dev and UAT servers using Static Assignments

### Ansible Refactoring and Static Assignments (IMPORTS AND ROLES)

In this project, I will be extending the functionality of this architecture (CI/CD and Configuration Management solution on the Development Servers using Ansible) and introducing configurations for UAT environment.

### STEP 1 - Jenkins Job Enhancement

I installed a plugin on Jenkins-Ansible server called COPY-ARTIFACTS.
![copyartifact](https://github.com/Oolabanji/test_/assets/136812420/50f91ec6-9de0-482f-9c68-f2f92470886d)

On the Jenkins-Ansible server, I created a new directory called ansible-config-artifact

'sudo mkdir /home/ubuntu/ansible-config-artifact'

Change permission of the directory

'chmod -R 0777 /home/ubuntu/ansible-config-artifact'

I created a new Freestyle project and named it save_artifacts.

![saveartifacts](https://github.com/Oolabanji/test_/assets/136812420/cc077871-4d10-4604-bd1a-c8f9eec1dbf6)

This project was triggered by completion of my existing ansible project and I configured it accordingly.

![artfactssetting](https://github.com/Oolabanji/test_/assets/136812420/ad5cbfde-86a0-445f-95d6-6bd59a11fb89)
![artifactsetting2](https://github.com/Oolabanji/test_/assets/136812420/7f894c98-9121-4f74-933c-ffcc1c679d00)
![artifactssetting3](https://github.com/Oolabanji/test_/assets/136812420/e734c0e0-edfc-4d58-9218-a2afe972f355)

I configured the number of build to 2. This is useful because whenever the jenkins pipeline runs, it creates a directory for the artifacts and it takes alot of space. By specifying the number of build, I can choose to keep only 2 of the latest builds and discard the rest.

I tested the set up by making some change in README.MD file inside the ansible-config-mgt repository (right inside main branch).


Now your Jenkins pipeline is more neat and clean.

![successartifact](https://github.com/Oolabanji/test_/assets/136812420/d38a7a4f-1383-415b-9247-2d4cecfbdf59)



![successartifact1](https://github.com/Oolabanji/test_/assets/136812420/57186747-1c68-4251-86fd-3e510238c3a8)


## Step 2 – Refactor Ansible code by importing other playbooks into site.yml


In playbooks folder, I created a new file and named it site.yml – This file will now be considered as an entry point into the entire infrastructure configuration.

I created a new folder in root of the repository and named it static-assignments. The static-assignments folder is where all other children playbooks will be stored

I moved common.yml file into the newly created static-assignments folder.

Inside site.yml file, I imported common.yml playbook.


![commondelyml](https://github.com/Oolabanji/test_/assets/136812420/8f1e179c-73b0-4849-970b-eae23347faad)

I run ansible-playbook command against the dev environment

I created another playbook under static-assignments and named it common-del.yml. In this playbook, I configured deletion of wireshark utility.

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





I updated site.yml with - import_playbook: ../static-assignments/common-del.yml instead of common.yml and run it against dev servers
cd /home/ubuntu/ansible-config-mgt/


'ansible-playbook -i inventory/dev.yml playbooks/site.yaml'

I made sure that wireshark is deleted on all the servers by running wireshark --version

![staticassign](https://github.com/Oolabanji/test_/assets/136812420/8dc23af3-108a-46a8-b3db-d9e69c9c9430)


![deletewireshark](https://github.com/Oolabanji/test_/assets/136812420/4ce401e4-c653-4302-9132-de895653b9a9)


### CONFIGURE UAT WEBSERVERS WITH A ROLE ‘WEBSERVER’

### Step 3 – Configure UAT Webservers with a role ‘Webserver’

I launched 2 fresh EC2 instances using RHEL 8 image, I used them as the uat servers(Web1-UAT and Web2-UAT).

![uatservers](https://github.com/Oolabanji/test_/assets/136812420/951b9634-8b91-46b3-b357-56328ce00e37)


I created a role using an Ansible utility called ansible-galaxy inside ansible-config-mgt/roles directory 

mkdir roles

cd roles

ansible-galaxy init webserver

After removing unnecessary directories and files, the roles structure looked like this.


└── webserver
    ├── README.md
    ├── defaults
    │   └── main.yml
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── tasks
    │   └── main.yml
    └── templates


   I updated my inventory ansible-config-mgt/inventory/uat.yml file with IP addresses of the 2 UAT Web servers

[uat-webservers]

172.31.29.3 ansible_ssh_user='ec2-user' 

172.31.30.221 ansible_ssh_user='ec2-user'

In /etc/ansible/ansible.cfg file, I uncommented roles_path string and provided a full path to the roles directory roles_path    = /home/ubuntu/ansible-config-mgt/roles, so Ansible could know where to find configured roles.

![rolepath](https://github.com/Oolabanji/test_/assets/136812420/a7a08184-f456-4eae-89e0-a630158b815a)


I installed and configured Apache (httpd service)
Clone Tooling website from GitHub https://github.com/Oolabanji/tooling.git.
I ensured the tooling website code was deployed to /var/www/html on each of 2 UAT Web servers and made sure httpd service is started.


---
- 'name: install apache

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

    repo: https://github.com/Oolabanji/tooling.git


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

    state: absent'


    ### REFERENCE WEBSERVER ROLE

    ### Step 4 – Reference ‘Webserver’ role

    In the static-assignments folder, I created a new assignment for uat-webservers 'uat-webservers.yml'.
Then I referenced the role.


---
-' hosts: uat-webservers

  roles:

     - webserver'

     Since the entry point to the ansible configuration is the site.yml file. So i referred the uat-webservers.yml role inside site.yml.


---
- hosts: all

- import_playbook: ../static-assignments/common.yml

- hosts: uat-webservers

- import_playbook: ../static-assignments/uat-webservers.yml


![siteyml](https://github.com/Oolabanji/test_/assets/136812420/f43056bd-b8a8-4d42-a037-d52a96cb9ed0)


![Screenshot 2023-09-18 092047](https://github.com/Oolabanji/test_/assets/136812420/2737f1b3-5f50-45d7-9097-47eb08106b20)

![Screenshot 2023-09-18 092021](https://github.com/Oolabanji/test_/assets/136812420/a99c9a7c-bdca-451d-8d5a-9964d5013737)


I committed my changes, created a Pull Request and merge them to main branch, made sure webhook triggered two consequent Jenkins jobs, they ran successfully and copied all the files to the Jenkins-Ansible server into /home/ubuntu/ansible-config-mgt/ directory.
Then I run playbook against my uat inventory.