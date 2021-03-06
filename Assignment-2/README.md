# DevOps Jenkins-Docker and Pipelines:

### (Pipeline & Job Chaining Project) :
____________________________________________________________________________________________________________________
### Prerequisites:

- YOU HAVE TO SET THE BUID TRIGGERS AND WEBHOOK AND DO CONFIGURATIONS IN JENKINS WHILE BUILDING JOBS.
- U SHOULD HAVE KNOWLEDGE ABOUT JENKINS CHAINING, HOW TO USE GITHUB PLUGIN TO INTEGRATE GITHUB, WEBHOOKS, SOME SHELL SCRIPTING KNOWLEDGE(LINUX COMMANDS), DOCKER KNOWLEDGE, GIT HOOKS.
----------------------------------

### Python Code:

- `ext_check.py`

```
import os
all_files=os.listdir("/storage")

all_htmls=[]
all_phps=[]

for file in all_files:
    if os.path.splitext(file)[1] ==".html" or os.path.splitext(file)[1]==".js":
        all_htmls.append(file)
    if os.path.splitext(file)[1] ==".php":
        all_phps.append(file)

if(len(all_htmls)==0 and len(all_phps)==0):
    os.system("tput setaf 3")
    print("NO FILE TO COPY")
    os.system("exit 0")
if len(all_htmls)>0:
    os.system("ssh root@192.168.29.62 rm -rf /htmltest_storatie/")
    os.system("ssh root@192.168.29.62 mkdir /htmltest_storage/")
    for htm in all_htmls:
        os.system("fscp {htm}  192.168.29.62:/htmltest_storage")
if len(all_phps)>0:
    os.system("ssh root@192.168.29.62 rm -rf /phptest_storage/")
    os.system("ssh root@192.168.29.62 mkdir /phptest_storage/")
    for php in all_phps:
        os.system(f"scp {php}  192.168.29.62:/phptest_storage")


print("TOTAL FILES COPIED:",len(all_files))

print("TOTAL PHP FILES  :",len(all_phps))
print("TOTAL HTML+JS FILES  :",len(all_htmls))
print("OTHER FILES:",len(all_files)-len(all_phps)-len(all_htmls))
```
_________________________________________________________________________________________________________

### Mailing Code:

- `mail.rc`

```
set smtp=smtps://smtp.gmail.com:465
set smtp-auth=login
set smtp-auth-user=YOUR@EMAIL
set smtp-auth-password=YOURPASSWORD
set ssl-verify=ignore
set nss-config-dir=/etc/pki/nssdb/
```
_________________________________________________________________________________________________________

### Step-1:
Create container image that’s has Jenkins installed using dockerfile.

### Dockerfile Code & Description:

```
FROM centos:latest
RUN yum install sudo -y
RUN yum install python3 -y
RUN yum install mailx -y
RUN yum install net-tools -y
RUN yum install wget -y
RUN yum install java-11-openjdk.x86_64 -y
RUN wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
RUN rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
RUN yum install openssh-clients -y
RUN yum install sudo -y
RUN yum install jenkins -y
RUN yum install /sbin/service -y
RUN  mkdir /workdir
COPY ./check_forext.py /workdir/
RUN mkdir /storage/
COPY  ./mail.rc  /etc/mail.rc
RUN echo -e "jenkins ALL(=ALL) NOPASSWD: ALL" >> /etc/sudoers
CMD java -jar /usr/lib/jenkins/jenkins.war
EXPOSE 8080
```

- FROM is used for the image to be used
- RUN is used for executing command while building the new image, so that features will be pre installed
- CMD  is used to run a command at the run time not at the build time and only one CMD should be there in one Dockerfile.
- Expose is used to expose the docker port as docker is isolated we have to enable Port Address Translation and is also used to inform  that which port is exposed to the other techonolgies or tools or commands.
- COPY is used to copy the file from the base os to image at build time.
- IN THE DIRECTORY WHERE YOU HAVE COPIED this,  THE `mail.rc` and  `ext_check.py` should be copied there only as per my defined Dockerfile.

- TO BUILD THE IMAGE USE THE COMMAND :
`docker build -t imagename:version` folder where the Dockerfile u have copied.
- For running the conatiner from this image use command:
`docker run -it -P --name containername imagename:version`
- YOU CAN ACCESS THE JENKINS FROM:
-base os using `BASEDOCKERHOSTIP:PORT MAPPED`.
- You can see the port number to which `8080` port of the container is mapped using `netstat -tnlp`.
_________________________________________________________________________________________________________

### Step-2:
When we launch this image, it should automatically starts Jenkins service in the container.
_________________________________________________________________________________________________________

### Step-3:
Create a job chain of job1, job2, job3, job4 and job5 using build pipeline plugin in Jenkins.
_________________________________________________________________________________________________________

### Job-1 Description:

To Pull the Github repo automatically when some developers push repo to Github.

 ```
#POST-COMMIT HOOK

#!/bin/bash

git push
```

- FIRST OF ALL NEED TO CONFIGURE JENKINS i.e. Putting the initial password , and enable `KEY-BASED AUTHENTICATION` with Remote host `ssh-keygen`. (on ssh client side i.e. our jenkin container)
- Copy this key to remote host ie in case our base os where docker is installed `ssh-copy-id root@IP`. (IP of ur baseos)  
- INORDER  TO PREVENT FROM ENTERING THE PASSWORD AGAIN AND AGAIN FOR REMOTE LOGIN.
- Install Github, BUILD PIPELINE, DELIVERY PIPELINE And GIT PULL REQUEST BUILDER plugins in JENKIN.
- One should have some knowledge of JENKINS, GITHUB AND GIT AND DOCKER.

### JENKINS JOB 1 SCRIPT:

```
rm -rvf /storage/
mkdir /storage/
cp -rvf * /storage/

python3 /workdir/check_forext.py
```
_________________________________________________________________________________________________________

### Job-2 Description:

By looking at the code or program file,

- Jenkins should automatically start the respective language interpreter install image container to deploy code.
- eg. If code is of PHP, then Jenkins should start the container that has PHP already installed.
- We can also use `"file -s filename"` to check for the content of the file also but i have created it for html js and php pages using different approach.

### JENKINS JOB 2 SCRIPT:

```
isempty_html=0
isempty_php=0                                                               #FIRST OF ALL NEED TO CREATE TWO DIRECTORIES  /htmltest_storage
if [[ $(ssh root@1IP  ls -A /htmltest_storage) ]]                            # and /phptest_storage
                                                                                #IP ==> BASE OS WHERE DOCKER IS INSTALLED
then
if ssh root@IP docker ps | grep test_html
then
ssh root@IP docker rm -f test_html
ssh root@IP docker run -dit -v /htmltest_storage:/usr/local/apache2/htdocs/ -p 8086:80 --name test_html httpd    #I HAVE USED httpd official image for html and js page deployment
else
ssh root@IP docker run -dit -v /htmltest_storage:/usr/local/apache2/htdocs/ -p 8086:80 --name test_html httpd
fi
else
if ssh root@IP docker ps | grep test_html
then
ssh root@IP docker rm -f test_html
echo EMPTY HTML FOLDER
fi
isempty_html=1
fi

if [[ $(ssh root@IP ls -A /phptest_storage) ]]      # to check for if directory is empty or not
then
if ssh root@IP docker ps | grep test_php
then
ssh root@IP docker rm -f test_php
ssh root@IP docker run -dit -v /phptest_storage:/var/www/html/ -p 8085:80 --name test_php vimal13/apache-webserver-php   #I HAVE USED docker image created by Vimal sir for php pages deployment
else
ssh root@IP docker run -dit -v /phptest_storage:/var/www/html/ -p 8085:80 --name test_php vimal13/apache-webserver-php
fi
else
if ssh root@IP docker ps | grep test_php
then
ssh root@IP docker rm -f test_php
echo EMPTY PHP FOLDER
fi
isempty_php=1
fi

if (( $isempty_html == 1 && $isempty_php == 1 ))
then

echo BOTH FOLDERS EMPTY
exit 1
fi
```
_________________________________________________________________________________________________________

### Job 3 & 4 Description:

- I HAVE COMBINED BOTH THE JOBS so that AFTER testing the Deployment is done if no issues.

Job3 : Test your app if it  is working or not.<br>
Job4 : if app is not working , then send email to developer with error messages.

- `test_php` and `test_html` are testing env.
- `prod_php` and `prod_html` are production env.
- I used `mailx` to send the mail I have done all the configurations in jenkins server regarding the mailx.

```
no_html=0
no_php=0

if ssh root@IP docker ps | grep test_html
then
for files in $(ssh root@IP ls -A /htmltest_storage);
do status=$(curl -o /dev/null -w "%{http_code}" -s IP:8086/${files}) ;
if [[ $status == 200 ]];
then echo -e SUCCESS CODE:  $status  "  "   FILENAME: $files  | mail -v -s "TESTING SUCCESS" RECEIVER-EMAILID  ;
else echo ERROR CODE: $status FILENAME: $files  | mail -v -s "TESTING FAILED" RECEIVER-EMAILID;no_html=1  
ssh root@IP docker rm -f test_html
fi
done
else
no_html=1
fi

if ssh root@IP docker ps | grep test_php
then
for file in $(ssh root@IP ls -A /phptest_storage);
do status=$(curl -o /dev/null -w "%{http_code}" -s IP:8085/${file}) ;
if [[ $status == 200 ]];
then echo -e SUCCESS CODE: $status "   " FILENAME: $file  | mail -v -s "TESTING SUCCESS" RECEIVER-EMAILID  ;
else
echo ERROR CODE: $status FILENAME: $file  | mail -v -s "TESTING FAILED" RECEIVER-EMAILID; no_php=1  ;
ssh root@IP docker rm -f test_php
fi
done
else
no_php=1
fi

if (( $no_php == 1 && $no_html == 1 ))
then
echo "All codes failed testing"  | mail -v -s "TESTING FAILED" RECEIVER-EMAILID
exit 1
else
if (( ( $no_php == 0 && $no_html == 0 )  || ( $no_php == 1 && $no_html == 0 ) || ( $no_php == 0 && $no_html == 1) ))
then
if (( $no_php == 0 ))
then
if ssh root@IP docker ps | grep test_php
then
if ssh root@IP docker ps | grep prod_php
then
ssh root@IP docker rm -f prod_php
ssh root@IP docker run -dit -v /phptest_storage:/var/www/html/ -p 8082:80 --name prod_php vimal13/apache-webserver-php
else
ssh root@IP docker run -dit -v /phptest_storage:/var/www/html/ -p 8082:80 --name prod_php vimal13/apache-webserver-php
fi
fi
fi
if (( $no_html == 0 ))
then
if ssh root@IP docker ps | grep test_html
then
if ssh root@IP docker ps | grep prod_html
then
ssh root@IP docker rm -f prod_html
ssh root@IP docker run -dit -v /htmltest_storage:/usr/local/apache2/htdocs/ -p 8081:80 --name prod_html httpd
else
ssh root@IP docker run -dit -v /htmltest_storage:/usr/local/apache2/htdocs/ -p 8081:80 --name prod_html httpd
fi
fi
fi
fi
fi
```
_________________________________________________________________________________________________________

### Job-5 Description: Job to monitor.

- If container where app is running, fails due to any reason then this job should automatically start the container again.

```
if ssh root@IP docker ps | grep test_php
then
if ssh root@IP docker ps | grep prod_php
then
echo "PHP SERVER UP AND RUNNING"
else
echo PHP Production Server Down  | mail -v -s "RELAUNCHING PHP SERVER"  RECEIVER-EMAILID
ssh root@IP docker run -dit -v /phptest_storage:/var/www/html/ -p 8082:80 --name prod_php httpd
fi
else

echo PHP Test Server Down  | mail -v -s " PHP SERVER DOWN"  RECEIVER-EMAILID
fi

if ssh root@IP docker ps | grep test_html
then
if ssh root@IP docker ps | grep prod_html
then
echo "HTML/STATIC Production Server UP AND RUNNING"
else
echo HTML/STATIC Pages Production Server Down  | mail -v -s "RELAUNCHING SERVER"  RECEIVER-EMAILID
ssh root@IP docker run -dit -v /htmltest_storage:/usr/local/apache2/htdocs/ -p 8081:80 --name prod_html httpd
fi
else

echo HTML Test Server Down  | mail -v -s " HTML SERVER DOWN " RECEIVER-EMAILID
fi
```
_________________________________________________________________________________________________________

<img src="https://www.threatstack.com/wp-content/uploads/2017/06/docker-cloud-twitter-card.png" align="center">


**Differences between Docker and VM:**

    Docker containers share the same system resources, they don’t have separate, dedicated hardware-level resources for them to behave like completely independent machines.
    They don’t need to have a full-blown OS inside.
    They allow running multiple workloads on the same OS, which allows efficient use of resources.
    Since they mostly include application-level dependencies, they are pretty lightweight and efficient. A machine where you can run 2 VMs, you can run tens of Docker containers without any trouble, which means fewer resources = less cost = less maintenance = happy people.

_______________________________________________________________________________________________________________
![new](https://messageconsulting.com/wp-content/uploads/2016/04/jenkins-technology-soup.png)
_______________________________________________________________________________________________________________

## Docker Commands:

    1. how to search a docker image in hub.docker.com

`docker search httpd`

    2. Download a docker image from hub.docker.com

`docker image pull <image_name>:<image_version/tag>`

    3. List out docker images from your local system

`docker image ls`

    4. Create/run/start a docker container from image

`docker run -d --name <container_Name> <image_name>:<image_version/tag>`

`d - run your container in back ground (detached)`

    5. Expose your application to host server

`docker run -d  -p <host_port>:<container_port> --name <container_Name> <image_name>:<Image_version/tag>`

`docker run -d --name httpd_server -p 8080:80 httpd:2.2`

    6. List out running containers

`docker ps`

    7. List out all docker container (running, stpooed, terminated, etc...)

`docker ps -a`

    8. run a OS based container which interactive mode (nothing but login to container after it is up and running)
    
```diff
docker run -i -t --name centos_server centos:latest
i - interactive
t - Terminal
```

    9. Stop a container

`docker stop <container_id>`

    10. Start a container which is in stopped or exit state

`docker start <container_id>`

    11. Remove a container

`docker rm <container_id>`

    12. login to a docker container

`docker exec -it <container_Name> /bin/bash`

_______________________________________________________________________________________________________________
![cloud-CI/CD](https://res.cloudinary.com/practicaldev/image/fetch/s--P2mKW4V3--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_66%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/zca7i3yizyfqyhzuocja.gif)
## Docker Install Apache Webserver - Dockerfile

Code to be written within Dockerfile:

```diff
FROM ubuntu:12.04

MAINTAINER Vedant Shrivastava vedantshrivastava466@gmail.com

LABEL version="1.1.0" \
      app_name="Docker Training application" \
	  release_date="1-January-2020"
RUN apt-get update && apt-get install -y apache2 && apt-get clean

ENV APACHE_RUN_USER www-data
ENV APACHE_RUN_GROUP www-data
ENV APACHE_LOG_DIR /var/log/apache2

EXPOSE 80

COPY index.html /var/www/html
CMD ["/usr/sbin/apache2", "-D", "FOREGROUND"]
```
_______________________________________________________________________________________________________________
## Jenkins:

**_'Jenkins to the rescue!'_**

As a Continuous Integration tool, Jenkins allows seamless, ongoing development, testing, and deployment of newly created code. Continuous Integration is a process wherein developers commit changes to source code from a shared repository, and all the changes to the source code are built continuously. This can occur multiple times daily. Each commit is continuously monitored by the CI Server, increasing the efficiency of code builds and verification. This removes the testers' burdens, permitting quicker integration and fewer wasted resources.


____________________________________________________________________________________________________________________________________
# Implementation and Understanding:

* Explanation and use case of Docker as a container based software to host programs such as any OS. 

* Introduction to concept of an environment which consists of WebServer and Distros.

* Launching of OS in isolation (locally) from docker.

* Defining OS Image as a bootable i/o system or device. 

* Also setup local webserver httpd from docker.
```diff
# docker pull httpd
# docker run -d -t -i --name Redhat_WebServer httpd
# docker cp new.html Redhat_WebServer
```

* Concept of mounting docker to remote developer's system explained.
```diff
# docker run -d -t -i /lwweb:/user/local/apache2/docs/ --name MyWebPage httpd
```

* Explanation of PAT networking to make local WebServer available to clients globally.
```diff
# docker run -d -t -i /lwweb:/user/local/apache2/docs/ -p 8001:80 --name ClientSideOpen httpd
```

* Integrating both Jenkins and Docker technologies along with the VCS such as Github with update data obtained from developers system.

* Explaining the necessity of two independent job creation for (1)obtaining code pushed to github by pull request by Jenkins, and (2)Deploying code to the environment.

* Explanation of the method of job chaining as build trigger , where one job depends on the others success, instead of simply being queued with it.

* Providing the root access to Jenkins using sudo command, after editing /etc/sudoers file, instead of the setfacl Access Control Listing command. 

_______________________________________________________________________________________________________________
# The Architecture:

1. The DEVELOPER & Local Workspace:

Each developer maintains a local workspace in personal system, and commits the changes/patches to Github through Git as and when necessary.

2. Github:

Once the changes are pushed to Github, the challenge is to have an automated architecture or system in order to deploy these changes to the webserver (after testing) in order for the clients to avail the benefits.

3. Jenkins:

This challenge is solved by Jenkins which serves as a middlemen to automate the deployment process.

________________________________________________________________________________________________________________________________________
# The Project:

<img src="https://i1.wp.com/www.docker.com/blog/wp-content/uploads/4fa92c35-5a00-4e7a-929e-e5ae4b99701a-1.jpg?w=1600&ssl=1" align="center">
________________________________________________________________________________________________________________________________________

## YAML Code for "Docker Automation Project" through Jenkins:

**Basic Project:**

```diff
---
- hosts: all
  become: true
  tasks:
  - name: stop if we have old docker container
    command: docker stop simple-devops-container
    ignore_errors: yes

  - name: remove stopped docker container
    command: docker rm simple-devops-container
    ignore_errors: yes

  - name: remove current docker image
    command: docker rmi simple-devops-image
    ignore_errors: yes
#    register: result
#    failed_when:
#      - result.rc == 0
#      - '"docker" not in result.stdout'


  - name: building docker image
    command: docker build -t simple-devops-image .
    args:
      chdir: /opt/docker

  - name: creating docker image
    command: docker run -d --name simple-devops-container -p 8080:8080 simple-devops-image
```

_______________________________________________________________________________________________________________
**Creating Docker Container:**

```diff
# Option-1 : Createting docker container using command module 
---
- hosts: all
  become: true

  tasks:
  - name: creating docker image using docker command
    command: docker run -d --name simple-devops-container -p 8080:8080 simple-devops-image
	
# option-2 : creating docker container using docker_container module 	
#  tasks:
#  - name: create simple-devops-container
#    docker_container:
#      name: simple-devops-container
#      image: simple-devops-image
#      state: present
#      recreate: yes
#      ports:
#        - "8080:8080"
```

_______________________________________________________________________________________________________________
**Creating Docker Image:**

```diff
# Option-1 : Createting docker image using command module 
---
- hosts: all
  become: true
  tasks:
  - name: building docker image
    command: docker build -t simple-devops-image .
    args:
      chdir: /opt/docker

# option-2 : creating docker image using docker_image module 

#  tasks:
#  - name: building docker image
#    docker_image:
#      build:
#        path: /opt/docker
#      name: simple-devops-image
#     tag: v1
#     source: build
```

_______________________________________________________________________________________________________________
## Jenkins Integration:

**JOB-1**
If Developer pushes to dev branch then Jenkins will fetch from dev and deploy on dev-docker environment.

**JOB-2**
If Developer pushes to master branch then Jenkins will fetch from master and deploy on master-docker environment.
both dev-docker and master-docker environment are on different docker containers.

**JOB-3**
Jenkins will check (test) for the website running in dev-docker environment. If it is running fine then Jenkins will merge the dev branch to master branch and trigger #job 2.

This is the repository where the Project has been uploaded.

_______________________________________________________________________________________________________________
### 1. Setting up the Host  System:

This is the system in which the developer works. We are considering it to be **Windows**. Here, Git is installed and authenticated. To make this project more realistic, we are going to work on 2 GitHub branches:
- master
- dev1

First, we create a new repository in GitHub. We copy the clone url of the repository and clone the it in the Desktop using `git clone <repo url>`.

We move inside the folder just created and create an `index.html` file.
Inside the file, lets type	<br>
`This is from master branch`
<br>
Then we add the file in the Staging Area using `git add . `<br>
Then we commit the changes  using the command: `git commit . -m "first commit from master"`<br>
Finally we push to GitHub using: `git push` <br>

We can also cross-check the contents from the GitHub website to confirm.

Now, we want to work with the `dev1` branch. We want to pull the contents from the `master` branch first.

To create and change the branch, we type: `git checkout -b dev1`
Now, all the codes from `master` branch are present in the `dev1` branch. We want to add an extra line in this branch:<br>
`cat >> This line is from Dev1 branch`

To stage, commit and push, we use the following commands:
```
git add . 
git commit . -m "Second commit from DEV1"
git push -u origin dev1
```

Now, we have setup 2 branches in Github, having some differential codes.

_______________________________________________________________________________________________________________
### 2. Setting up the Server System:

This is the system where the Web Server will be running. We are going to use Docker to run 2 `httpd` servers, one for deploying the `master` branch and the other for the `dev1` branch. We are going to use RHEL 8 as the Operating System to host the Docker containers.<br>

As we are going to automate the process, we are going to configure Jenkins for creating the CI/CD pipeline.<br>

**Jenkins** would do 2 important tasks for each of the branches:
- Dowloading any code changes from GitHub to the server system.
- Starting the Webserver from Docker containers, to view the final website.

**Jenkins** would do another important task from the **Quality Assurance Team**. If the `dev1` branch is working fine, then Jenkins would merge it with the `master` branch, on being triggered to do so.

We create 2 directories on our Desktop directory from the terminal.
- lwtest
- lwtestdev

The codes downloaded from **GitHub** by **Jenkins** would be placed in these folders according to their branches.

Now, lets configure Jenkins to do all those tasks.

_______________________________________________________________________________________________________________
### 3. Configuring Jenkins:

Lets start the Jenkins process from RHEL 8 by the command: `systemctl start jenkins`<br>
Now, on visiting the port **8080**, we can see the Jenkins Control Panel running.

But, we are going to configure it from Host / Developer system by getting the IP address of RHEL. <br> For that we use the command `ifconfig enp0s3`

After getting the IP Address, from any browser of the **Host System**, we can direct to the IP address.<br>
for e.g. `http://192.123.32.2932:8080` <br>
After logging in with both the username and password as `admin`, we come to the Jobs List page. Here we can see the Jobs we have submitted to be performed.

#### 1. Automatic Code Download:

For now, we have to create new Jobs for downloading the latest codes from **both** the branches of Github separately, to the Server system, for being deployed on the Web-server.<br>
i.e. The `master` branch should be downloaded in the `lwtest` folder and the `dev1` branch would be downloaded in the `lwtestdev` folder.

For creating the Job for downloading codes from the **master** branch:
1. Select `new item` option from the Jenkins menu. 
2. Assign a name to the Job ( eg. lwtest-1 )and select it to be a `Freestyle` project.
3. From the `Configure Job` option, we set the configurations.
4. From **Source Code Management** section, we select Git and mention the URL of our GitHub Repository and select the branch as `master`.
5. In the **Build Triggers** section, we select `Poll SCM` and set the value to `* * * * *`.<br> **This means that the Job would check any code change from GitHub every minute.**
6. In the Build Section, we type the following script:
			`sudo cp -v -r -f * /root/Desktop/lwtest`
   **This command would copy all the content downloaded from the GitHub master branch to the specified folder for deployment.**
7. On clicking the **Save** option, we add the Job to our Job List.

On coming back to the Job List page, we can see the **Job** is being built. If the color of the ball turns **blue**, it means the Job has been successfully excecuted. If the color changes to **red**, it means there has been some error in between. We can see the console output to check the error.

**The same process can be performed to create another Job `lwtest-2` for the `dev1` branch. Only the branch name needs to be changed to `dev1` and the script:**<br>`sudo cp -v -r -f * /root/Desktop/lwtestdev`

Till now, we have successfully downloaded the codes from both the branches of GitHub to our Server System automatically.


#### 2. Automatically Starting the Docker containers:


Next, we create another Job for starting the docker containers once the codes have been copied into the `lwtest` and `lwtestdev` folders.

For starting the docker container once the `lwtest` folder updates from the **master** branch:
1.  Select `new item` option from the Jenkins menu. 
2.  Assign a name to the Job ( eg. lwtest-1-docker )and select it to be a `Freestyle` project.
3.  From the `Configure Job` option, we set the configurations.
4.  From **Build Triggers** section, we select `Build after other projects are built` and mention `lwtest` as the project to watch. This is called **DownStreaming**.
5.  In the **Build** Section, we type the following script:
```
if sudo docker ps | grep docker_master
then
echo "Already Running"
else
sudo docker run -d  -t -i -p 8085:80 -v /root/Desktop/lwtest:/usr/local/apache2/htdocs/ --name docker_master httpd
fi
```

This script first checks if any Docker container has been already created, in such case it exits. <br>
Else, it starts a new Docker container of the `httpd` webserver, with some parameters:
- Runs the container in background : `-d`
- Exposes the port so that the container can be operated from outside the Server system (RHEL)
- Mounts the `lwtest` folder on the `htdocs` folder, from where the webserver renders the pages.
- Sets the name of the container as `docker_master`
6. On clicking the **Save** option, we add the Job to our Job List.

**The same process can be performed to create another Job `lwtest-2-docker` for starting the docker containers once the `lwtestdev` folder updates from the `dev1` branch.**<br> **Only the build trigger needs to be set to watch `lwtestdev` and the corresponding folder name should be changed in the script.**

These 2 Jobs would automatically run just after the codes are updated in the respective folders, by the previous jobs.

Thus we have successfully set up a **CI/CD pipeline** where once the developer pushes the code on GitHub, **Jenkins** would automatically download the codes and start the **Docker containers** to run the Web-Server.


#### 3. Merging the `dev1` branch with `master` by QAT:

Finally, this is the task of the **Quality Assurance Team**. When the team certifies that the `dev1` branch is working fine, they can merge it with the `master` branch using **Remote Build Triggers**.

To setup the Trigger we are going to use **Jenkins** to do the automation:
1. Select **new item** option from the Jenkins menu. 
2. Assign a name to the Job ( eg. Merge-test )and select it to be a **Freestyle** project.
3. From the **Configure Job** option, we set the configurations.
4. From **Source Code Management** section, we select Git and mention the URL of our GitHub Repository and select the branch to build as `dev1`.
5. From the **Additional Behavior** Dropdown list, we select `Merge Before Build`
6. There, we set the **Name of the Repository** as `origin` 
7. We set the **Branch to Merge to** as `master`
8. From the **Build Triggers** section, we select **Trigger builds remotely** option.
9. Provide an **Authentication Token**
10. From the **Post Build Actions** dropdown, we select **Git Publisher**.
11. We check **Push only if build succeeds** and **Merge Results** options.
12.  On clicking the **Save** option, we add the Job to our Job List.
13.  Now from **Build Triggers** section of the **lwtest** job, we select `Build after other projects are built` and mention `Merge-test` as the project to watch.

Thus, the Job has been setup. To Trigger the Build, the **QAT** would run the following command:<br>
`curl --user "<username>:<password>" JENKINS_URL/job/Mergetest/build?token=TOKEN_NAME`

e.g. `curl --user "admin:admin" http://192.123.32.2932:8080/job/Merge-test/build?token=redhat`

So, after merging, the **lwtest** Job is again fired, so that the updated code can be downloaded in the Server system to run in the web-server.

_________________________________________________________________________________________________________

<img src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2019/10/20/Diagram2.png" align="center">

<img src="https://www.threatstack.com/wp-content/uploads/2017/06/docker-cloud-twitter-card.png" align="center">

_________________________________________________________________________________________________________

**Differences between Docker and VM:**

    Docker containers share the same system resources, they don’t have separate, dedicated hardware-level resources for them to behave like completely independent machines.
    They don’t need to have a full-blown OS inside.
    They allow running multiple workloads on the same OS, which allows efficient use of resources.
    Since they mostly include application-level dependencies, they are pretty lightweight and efficient. A machine where you can run 2 VMs, you can run tens of Docker containers without any trouble, which means fewer resources = less cost = less maintenance = happy people.

_______________________________________________________________________________________________________________
![CI/CD_Process](https://raw.githubusercontent.com/sumitc91/data/master/askgif-blog/8f5fb6c4-3070-4ce0-a3a9-ea0070a3fc7c_docker.jpg)
_______________________________________________________________________________________________________________

## Docker Commands:

    1. how to search a docker image in hub.docker.com

`docker search httpd`

    2. Download a docker image from hub.docker.com

`docker image pull <image_name>:<image_version/tag>`

    3. List out docker images from your local system

`docker image ls`

    4. Create/run/start a docker container from image

`docker run -d --name <container_Name> <image_name>:<image_version/tag>`

`d - run your container in back ground (detached)`

    5. Expose your application to host server

`docker run -d  -p <host_port>:<container_port> --name <container_Name> <image_name>:<Image_version/tag>`

`docker run -d --name httpd_server -p 8080:80 httpd:2.2`

    6. List out running containers

`docker ps`

    7. List out all docker container (running, stpooed, terminated, etc...)

`docker ps -a`

    8. run a OS based container which interactive mode (nothing but login to container after it is up and running)
    
```diff
docker run -i -t --name centos_server centos:latest
i - interactive
t - Terminal
```

    9. Stop a container

`docker stop <container_id>`

    10. Start a container which is in stopped or exit state

`docker start <container_id>`

    11. Remove a container

`docker rm <container_id>`

    12. login to a docker container

`docker exec -it <container_Name> /bin/bash`

_______________________________________________________________________________________________________________
## Docker Install Apache Webserver - Dockerfile

Code to be written within Dockerfile:

```diff
FROM ubuntu:12.04

MAINTAINER Vedant Shrivastava vedantshrivastava466@gmail.com

LABEL version="1.1.0" \
      app_name="Docker Training application" \
	  release_date="1-January-2020"
RUN apt-get update && apt-get install -y apache2 && apt-get clean

ENV APACHE_RUN_USER www-data
ENV APACHE_RUN_GROUP www-data
ENV APACHE_LOG_DIR /var/log/apache2

EXPOSE 80

COPY index.html /var/www/html
CMD ["/usr/sbin/apache2", "-D", "FOREGROUND"]
```
_______________________________________________________________________________________________________________
## Jenkins:
# Implementation and Understanding:

* Explanation and use case of Docker as a container based software to host programs such as any OS. 

* Introduction to concept of an environment which consists of WebServer and Distros.

* Launching of OS in isolation (locally) from docker.

* Defining OS Image as a bootable i/o system or device. 

* Also setup local webserver httpd from docker.
```diff
# docker pull httpd
# docker run -d -t -i --name Redhat_WebServer httpd
# docker cp new.html Redhat_WebServer
```

* Concept of mounting docker to remote developer's system explained.
```diff
# docker run -d -t -i /lwweb:/user/local/apache2/docs/ --name MyWebPage httpd
```

* Explanation of PAT networking to make local WebServer available to clients globally.
```diff
# docker run -d -t -i /lwweb:/user/local/apache2/docs/ -p 8001:80 --name ClientSideOpen httpd
```

* Integrating both Jenkins and Docker technologies along with the VCS such as Github with update data obtained from developers system.

* Explaining the necessity of two independent job creation for (1)obtaining code pushed to github by pull request by Jenkins, and (2)Deploying code to the environment.

* Explanation of the method of job chaining as build trigger , where one job depends on the others success, instead of simply being queued with it.

* Providing the root access to Jenkins using sudo command, after editing /etc/sudoers file, instead of the setfacl Access Control Listing command. 

_______________________________________________________________________________________________________________
# The Architecture:

1. The DEVELOPER & Local Workspace:

Each developer maintains a local workspace in personal system, and commits the changes/patches to Github through Git as and when necessary.

2. Github:

Once the changes are pushed to Github, the challenge is to have an automated architecture or system in order to deploy these changes to the webserver (after testing) in order for the clients to avail the benefits.

3. Jenkins:

This challenge is solved by Jenkins which serves as a middlemen to automate the deployment process.

________________________________________________________________________________________________________________________________________
# The Project:

<img src="https://i1.wp.com/www.docker.com/blog/wp-content/uploads/4fa92c35-5a00-4e7a-929e-e5ae4b99701a-1.jpg?w=1600&ssl=1" align="center">
________________________________________________________________________________________________________________________________________

## YAML Code for "Docker Automation Project" through Jenkins:

**Basic Project:**

```diff
---
- hosts: all
  become: true
  tasks:
  - name: stop if we have old docker container
    command: docker stop simple-devops-container
    ignore_errors: yes

  - name: remove stopped docker container
    command: docker rm simple-devops-container
    ignore_errors: yes

  - name: remove current docker image
    command: docker rmi simple-devops-image
    ignore_errors: yes
#    register: result
#    failed_when:
#      - result.rc == 0
#      - '"docker" not in result.stdout'


  - name: building docker image
    command: docker build -t simple-devops-image .
    args:
      chdir: /opt/docker

  - name: creating docker image
    command: docker run -d --name simple-devops-container -p 8080:8080 simple-devops-image
```

_______________________________________________________________________________________________________________
**Creating Docker Container:**

```diff
# Option-1 : Createting docker container using command module 
---
- hosts: all
  become: true

  tasks:
  - name: creating docker image using docker command
    command: docker run -d --name simple-devops-container -p 8080:8080 simple-devops-image
	
# option-2 : creating docker container using docker_container module 	
#  tasks:
#  - name: create simple-devops-container
#    docker_container:
#      name: simple-devops-container
#      image: simple-devops-image
#      state: present
#      recreate: yes
#      ports:
#        - "8080:8080"
```

_______________________________________________________________________________________________________________
**Creating Docker Image:**

```diff
# Option-1 : Createting docker image using command module 
---
- hosts: all
  become: true
  tasks:
  - name: building docker image
    command: docker build -t simple-devops-image .
    args:
      chdir: /opt/docker

# option-2 : creating docker image using docker_image module 

#  tasks:
#  - name: building docker image
#    docker_image:
#      build:
#        path: /opt/docker
#      name: simple-devops-image
#     tag: v1
#     source: build
```

_______________________________________________________________________________________________________________

### Author:
----------------------------------
```diff
! Anmol Sinha | 1805553@kiit.ac.in
```
