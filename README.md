## Integrating Docker in CI/CD Pipeline

### Step1: Setup Docker Environment

- First we will create an EC2 instance for Docker and name it as `Docker-Host`. 
```sh
instanceType: t2.micro
AMI: Amazon Linux-2
Security Group: 
22, SSH
8080, Custom TCP
```

- Login to `Docker-Host` server via SSH,  switch to root user `sudo su -` and install docker
```sh
yum install docker -y
systemctl start docker
```

### Step2: Create Tomcat container

- Go to `DockerHub`, search for `Tomcat` official image. We can pick a specific tag of the image, but if we don't specify it will pull the image with latest tag. For this project we will use latest Tomcat image.
```sh
docker pull tomcat
```

- Next we will run docker container from this image.
```sh
docker run -d --name tomcat-container -p 8081:8080 tomcat
```

- To be able to reach this container from browser, we need to add port `8081` to our Security Group. We can add a range of port numbers `8081-9000` to Ingress.

- Once we try to reach our container from browser it will give us `404 Not found` error. This is a known issue after Tomcat version 9. We will follow below steps to resolve this issue.

- First we need to go inside container by running below command:
```sh
docker exec -it tomcat-container /bin/bash
```  

- Once we are inside the container, we need to move files under `webapps.dist/` to `webapps/`
```sh
mv webapps webapps2
mv webapps.dist webapps
exit
```

- However, this solution is temporary. Whenever we stop our container and restart the same error will be appeared. To overcome this issue, we will create a Dockerfile and create our own Tomcat image.

### Step3: Create Customized Dockerfile for Tomcat

- We will create a Dockerfile which will fix the issue.

```sh
FROM tomcat
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps/
```

- Lets create an image from this Docker file.
```sh
docker build -t tomcat-fixed .
```

- Now we can create a container from this image
```sh
docker run -d --name tomcat-container-fixed -p 8085:8080 tomcat-fixed
```

- We can check our Tomcat server in browser now.

![app-v2](app-v2-deployed-to-tomcat.png)
### Step4: Integrate Docker with Jenkins

- We will create a new user/password called `dockeradmin` and add it to `docker` group

```sh
useradd dockeradmin
passwd dockeradmin
usermod -aG docker dockeradmin
```
- Start docker service.
```sh
systemctl status docker
systemctl start docker
systemctl enable docker
systemctl status docker
```
- We can check our new user in `/etc/passwd` and groups `/etc/group`.
```sh
cat /etc/passwd
cat /etc/group
```

- Next we will allow username/password authentication to login to our EC2 instance. By default, EC2s are only allowing connection with Keypair via SSH not username/password.

```sh
vim /etc/ssh/sshd_config
```

- We need to uncomment/comment below lines in `sshd_config` file and save it
```sh
PasswordAuthentication yes
#PasswordAuthentication no
```
- We need to restart sshd service
```sh
service sshd reload
```

- Go to Jenkins , install `Publish over SSH` plugin. next go to `Manage Jenkins` -> `Configure System`
Find `Publish over SSH` -> `SSH Server`. `Apply` changes and `Save`
```sh
Name: dockerhost
Hostname: Private IP of Docker Host(since Jenkins and Docker host are in the same VPC, they would be able to communicate over same network)
Username: dockeradmin
click Advanced
Check use password based authentication
provide password
```

- We will create a Jenkins job with below properties:
```sh
Name: BuildAndDeployOnContainer-CI
Type: Maven Project
SCM: https://github.com/nileshlip/hello-world-Projects.git
POLL SCM: * * * * *
Build Goals: clean install
Post build actions: Send build artifacts over ssh
SSH server: dockerhost
TransferSet: **/*.war
Remove prefix: target
Remote directory://opt//docker                  (/home/dockeradmin)
```

### Step5: Update Tomcat dockerfile to automate deployment process

- Currently artifacts created through Jenkins are sent to `/home/dockeradmin`. We would like to change it to home directory of root user, and give ownership of this new directory to `dockeradmin`

```sh
sudo su - 
cd /opt
sudo mkdir docker
sudo chown -R dockeradmin:dockeradmin docker
```

- We have our Dockerfile under `/home/root`, we will move Dockerfile under `docker` directory and change ownership as well 
```sh
sudo mv Dockerfile /opt/docker
sudo chown -R dockeradmin:dockeradmin docker
```

- We will change our Dockerfile to copy the webapps.war file under webapps/ in Tomcat container.

```sh
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps/
COPY ./*.war /usr/local/tomcat/webapps
```
 
- we build the image and create a container from the newly created image.
```sh
docker build -t tomcat:v1 .
docker run -d --name tomcatv1 -p 8086:8080 tomcat:v1
```
- We can check our app from browser http://<public_ip_of_docker_host>:8086/webapp/

- Now we can configure our `BuildAndDeployOnContainer` job to deploy our application. We will add below lines to `Exec Command` part. We also need to change `Remote directory` path as `//opt//docker`

```sh
cd /opt/docker;
docker build -t regapp:v1 .;
docker stop registerapp;
docker rm registerapp;
docker run -d --name registerapp -p 8089:8080 regapp:v1
```
