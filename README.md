
# jenkins-ansible-docker-git-project
This is a DevOps jenkins freestyle project using Git, Jenkins, Ansible and Docker on AWS for deploying a python-flask application in a Docker container.The process should be initiated from a new commit to a specific branch of a GitHub repository. This event kicks off a process that begins building the Docker image. Jenkins supports this event-driven flow using the “GitHub hook trigger for GITScm polling".

# Architecture

![158003817-bb60b4aa-5aac-4bb0-9a12-f9f857c7226a](https://user-images.githubusercontent.com/98936958/167253171-1145d1a1-7057-4d54-b165-6707c7ab7705.png)

 1. jenkins Server
 2. Docker Image Build Server
 3. Production/Test Server

## Installing / Configuring Jenkins, Git and Ansible
~~~sh
amazon-linux-extras install epel -y
yum install java-1.8.0-openjdk-devel -y
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
yum install jenkins git ansible -y
systemctl start jenkins.service
systemctl enable jenkins.service
~~~

After the instalation, we can test the jenkins using the below port:
~~~sh
http://3.110.194.33:8080/   //provide your jenkins server Public IP
~~~

## Ansible Inventory file
~~~sh
[buildserver]
172.31.13.7 ansible_user="ec2-user"

[testserver]
172.31.4.79 ansible_user="ec2-user"
~~~

## Variable files

# cat packagevariable.yml
~~~sh
---

packages:
  - git
  - pip
  - docker
~~~

# cat variables.yml
~~~sh
---

git_repo_url: "https://github.com/JuliousGonsalves/devops-flask.git"
clone_directory: "/var/flaskapp/"
docker_login_user: "juliousgonsalves"   // Use you docker Hub user name
docker_login_pass: "password"         // user your docker password
docker_image: "juliousgonsalves/flaskapplication"
~~~

## Ansible Playbook

# cat main.yml
~~~sh
---

- name: "Building docker image on a build serveri using git repo"
  hosts: buildserver
  become: true
  vars_files:
    - packagevariable.yml
    - variables.yml

  tasks:

    - name: "Installing docker, Git and pip - BuildServer"
      yum:
        name: "{{packages}}"
        state: present

    - name: "Adding ec2-user to docker group - BuildServer"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: yes

    - name: "Installing docker-py python pacakge for docker using pip -BuildServer"
      pip:
        name: docker-py

    - name: "Starting/Enabling Docker service - BuildServer"
      service:
        name: "docker"
        state: restarted
        enabled: true

    - name: "Cloning image using git to {{clone_directory}} - BuildServer"
      git:
        repo: "{{git_repo_url}}"
        dest: "{{clone_directory}}"
      register: cloning_status

    - debug:
        msg: "{{cloning_status}}"


    - name: "Login to dockerHub - BuildServer"
      when: cloning_status.changed == true
      docker_login:
        username: "{{docker_login_user}}"
        password: "{{docker_login_pass}}"
        state: present

    - name: "Creating Docker image and pushing to docker hub - BuildServer"
      when: cloning_status.changed == true
      docker_image:
        source: build
        build:
          path: "{{clone_directory}}"
          pull: yes
        name: "{{docker_image}}"
        tag: "{{item}}"
        push: yes
        force_tag: yes
        force_source: yes
      with_items:
        - "{{cloning_status.after}}"
        - latest

    - name: "Removing the image from local sytem"
      when: cloning_status.changed == true
      docker_image:
        state: absent
        name: "{{docker_image}}"
        tag: "{{item}}"
      with_items:
        - "{{cloning_status.after}}"
        - latest

    - name: "Logout to dockerHub - BuildServer"
      when: cloning_status.changed == true
      docker_login:
        username: "{{docker_login_user}}"
        password: "{{docker_login_pass}}"
        state: absent



- name: "Running container on Test Instance"
  become: true
  hosts: testserver
  vars:
    packages :
      - docker
      - pip
    docker_image: "juliousgonsalves/flaskapplication"
  tasks:

    - name: "Installing docker and  pip module in test server"
      yum:
        name: "{{packages}}"
        state: present

    - name: "Adding ec2-user to docker group - test Server"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: yes

    - name: "Installing docker-py python pacakge for docker using pip -test Server"
      pip:
        name: docker-py

    - name: "Starting/Enabling Docker service - test Server"
      service:
        name: "docker"
        state: restarted
        enabled: true


    - name: "pulling image to test server"
      docker_image:
        name: "{{docker_image}}"
        source: pull
        force_source: yes
      register: dockerimage_status

    - debug:
        msg: "{{dockerimage_status}}"

    - name: "running container in test server"
      when: dockerimage_status.changed == true
      docker_container:
        name: flaskapplicaton
        image: "{{docker_image}}:latest"
        recreate: yes
        pull: yes
        published_ports:
          - "80:8080"
~~~

## Docker logins is encrypted using ansible vault

~~~sh
ansible-vault encrypt /var/deployment/variables.yml
New Vault password:
Confirm New Vault password:
Encryption successful
~~~

## Configuring Jenkins

1. Login to Jenkins.
   http://3.110.194.33:8080/
   password : /var/lib/jenkins/secrets/initialAdminPassword ( Initial password can be obtained from this location)
   
   ![jenkinverification](https://user-images.githubusercontent.com/98936958/167253162-b81ba130-2f44-450b-ae2d-07809df86dae.PNG)
   
   
2. Customise Plugins >> Install suggested Plugins

3. Create Jenkin admin user

![jenkinadminuser](https://user-images.githubusercontent.com/98936958/167253161-d82d2aaa-23d9-49c9-82fa-5d497ca9f9e8.PNG)

4. Install Ansible plugin 
   
   Manage Jenkins >> Plugin Manager  >> Ansible >> Install without restart
   
   ![ansibleplugininstall](https://user-images.githubusercontent.com/98936958/167253158-7c993f60-e73b-499a-a78c-203d332aac92.PNG)
   
   Once the plugin installation is done you will need to restart jenkins.

5. Go to  Manage Jenkins >> Global Tool Configuration >> Under Ansible [ Name : Ansible Path to ansible executables directory: /bin/]

6. To create  a job

   New item >> Enter Item Name >> Freestyle Project >> OK
   
   ![freestyle](https://user-images.githubusercontent.com/98936958/167253939-8ac88f71-cd3f-4e28-88b0-ea7ea57cef8c.PNG)

   Add a decription  "Docker Image Build and test project" >> Source Code Management select Git and add repo url  >> Under Build, choose 
   "Invoke ansible Playbook", then add Playbook path "/var/deployment/main.yml".

   Add Inventory >> File or host list >> File path or comma separated host list "/var/deployment/hosts"

   Under advanced section, make sure Disable the host SSH key check is disabled.


7. Under Vault Credentials >> add Jenkins >> refer screnshot to add vault logins >> Then under Vault credential choose "Vault password"

   ![vaultlogins](https://user-images.githubusercontent.com/98936958/167253163-fb481d17-d8b7-4c26-aed9-caecdf4b37a1.PNG)  

8. Add ssh details in Credentials.

  ![sshdetails](https://user-images.githubusercontent.com/98936958/167253773-fc975685-d330-4f10-aaad-55f050cecd90.PNG)
  
  Add Kind as "SSH username / private key"  , User name "ec2-user" and private key "you ssh pem file"
  
  ![sshkey](https://user-images.githubusercontent.com/98936958/167253775-bdbda71d-ab9b-457c-a913-48027422047b.PNG)

9. Manually run Jenkins Job and verify it is working fine. Ie, Select Job and click on "Build now"

   ![manually](https://user-images.githubusercontent.com/98936958/167254110-08267cf3-3fda-45ce-b160-8cd5a0b56cb6.PNG)

## Automating the build and test process using git hub hooks

   In order to do this we need to configure jenkins payload url in Git repository

1. Go to your Git repo , under security >> choose Webhooks

   ![webhook](https://user-images.githubusercontent.com/98936958/167254245-38c90d90-6f11-49c3-acdc-5ecc10f40e6a.PNG)

2. Acces your Jenkin Admin dashboard >> Select your Jon and click on "Configure" and select "GitHub hook trigger for GITScm polling"

  ![buildgit](https://user-images.githubusercontent.com/98936958/167254309-5efb73ad-da4a-437b-a952-dba884587045.PNG)


# Result

Once the developer make changes in git repo, An alert is triggered based on it Jenkins run the job.
ie A new image is pushed to Docker Hub from the 'Build' host, which is pulled by 'Test' host to create a Docker Container.

![finaloupt](https://user-images.githubusercontent.com/98936958/167253094-950fc795-4eb6-4e0a-ada6-c78f6bb11685.PNG)




