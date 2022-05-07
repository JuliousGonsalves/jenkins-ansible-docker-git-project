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
docker_login_user: "juliousgonsalves"
docker_login_pass: "password"
docker_image: "juliousgonsalves/flaskapplication"
~~~








![ansibleplugininstall](https://user-images.githubusercontent.com/98936958/167253158-7c993f60-e73b-499a-a78c-203d332aac92.PNG)
![finaloupt](https://user-images.githubusercontent.com/98936958/167253160-c5a0f384-28ba-470f-baef-da3e1f559db7.PNG)
![jenkinadminuser](https://user-images.githubusercontent.com/98936958/167253161-d82d2aaa-23d9-49c9-82fa-5d497ca9f9e8.PNG)
![jenkinverification](https://user-images.githubusercontent.com/98936958/167253162-b81ba130-2f44-450b-ae2d-07809df86dae.PNG)
![vaultlogins](https://user-images.githubusercontent.com/98936958/167253163-fb481d17-d8b7-4c26-aed9-caecdf4b37a1.PNG)
![finaloupt](https://user-images.githubusercontent.com/98936958/167253094-950fc795-4eb6-4e0a-ada6-c78f6bb11685.PNG)
