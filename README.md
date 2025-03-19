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

### Step2: Integrate Docker with Jenkins

- We will create a new user/password called `dockeradmin` and add it to `docker` group

```sh
useradd dockeradmin
passwd dockeradmin
usermod -aG docker dockeradmin
su - dockeradmin  
```
- Open `visudo` and add `dockeradmin` For Root Previlege
- Start docker service.
```sh
systemctl status docker
systemctl start docker
systemctl enable docker
systemctl status docker
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
- Go to Jenkins , install `Publish over SSH` plugin. next go to `Manage Jenkins` -> `System`
Find  `SSH Server`. `Apply` changes and `Save`
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
SCM: https://github.com/nileshlip/
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
- Configure tomcat-users.xml for Manager Access ,Create (tomcat-users.xml) File On docker in /opt/docker/
```sh
  <tomcat-users>
  <!-- Define roles -->
  <role rolename="manager-gui"/>
  <role rolename="manager-script"/>
  <role rolename="manager-status"/>

  <!-- Define users with appropriate roles -->
  <user username="admin" password="admin123" roles="manager-gui,manager-status"/>
  <user username="deployer" password="deploy123" roles="manager-script"/>
</tomcat-users>
```
- Modify context.xml to Allow Remote Access ,Create (context.xml) file on docker In /opt/docker/  
```sh
 <Context antiResourceLocking="false" privileged="true">
    <!-- Allow access from any IP address -->
    <Valve className="org.apache.catalina.valves.RemoteAddrValve"
           allow=".*" />
</Context>
```   
- We will create same Dockerfile under `docker` directory in Ansible host,you can create image and run container from this image in `ansible` server.
```sh
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps/
COPY ./*.war /usr/local/tomcat/webapps
COPY tomcat-users.xml /usr/local/tomcat/conf/
COPY context.xml /usr/local/tomcat/webapps/manager/META-INF/
```
- try to build image for test purpose
```sh
docker build -t regapp:v1 .
docker run -t --name regapp-server -p 8081:8080 regapp:v1
```
- We can check our app from browser http://<public_ip_of_docker_host>:8081/webapp/

- Now we can configure our `BuildAndDeployOnContainer` job to deploy our application. We will add below lines to `Exec Command` part. We also need to change `Remote directory` path as `//opt//docker`

```sh
cd /opt/docker;
docker build -t regapp:v1 .;
docker stop registerapp;
docker rm registerapp;
docker run -d --name registerapp -p 8089:8080 regapp:v1
```
