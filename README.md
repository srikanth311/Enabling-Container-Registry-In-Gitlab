### Step 1: Create an EC2 instance with Gitlab Community Edition selected from Market place as shown in the below image.
![Alt text](Images/mp.png?raw=true "Gitlab community edition selection")

### Step 2: Make sure your ssh port(22) and http and https ports (80 and 443) and ICMP are opened in Inbound rule of the ec2 instance's security group. Check the below image.

![Alt text](Images/sg.png?raw=true "Security groups")

### Step 3: Get the public IP address of the ec2 instance that was created above and opened it in the browser to connect to gitlab UI.

#### In the browser type the IP address: `http://xx.xx.xx.xx:80`. Change the IP address accordingly.
#### When you first connect to this, it will ask you to change the password, go ahead and change the password.
#### Then you can sign in: Username is : root and Password is: the one you just used to update it.

![Alt text](Images/login.png?raw=true "Login")

### Step 4:
#### By default the container registry is not enabled or setup. Follow the below steps to set it up.

### Step 5: Login to EC2 instance that was created above.
`ssh -i ~/aws-key-pair1.pem ubuntu@xx.xx.xx.xx`

### Step 6: Creating SSL certificates
#### We will be using open ssl for docker registry URL.
#### Create a folder to store the ssl certificates.
```
> sudo chmod -R 777 /etc/gitlab
> cd /etc/gitlab/
> sudo chmod -R 755 /etc/gitlab/trusted-certs/
> cd trusted-certs/
> sudo openssl req -newkey rsa:4096 -nodes -sha256 -keyout /etc/gitlab/trusted-certs/registry.srikanth.com.key -x509 -days 365 -out /etc/gitlab/trusted-certs/registry.srikanth.com.crt

```
```
Note: When prompted, make sure you provide the correct "FQDN" for the common name. In my case, i have provided "registry.srikanth.com"
```

```
O/P:
====

9 -days 365 -out /etc/gitlab/trusted-certs/registry.srikanth.com.crt
Generating a 4096 bit RSA private key
.................................................................................................++
...............................................................................++
writing new private key to '/etc/gitlab/trusted-certs/registry.srikanth.com.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:US
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:registry.srikanth.com
Email Address []:

```

#### It will create two files in the `/etc/gitlab/trusted-certs` directory.
```
Change the permissions of the files that were created above.

chmod 600 /etc/gitlab/trusted-certs/registry.srikanth.com.crt
chmod 600 /etc/gitlab/trusted-certs/registry.srikanth.com.key
```

### Step 7: Update the gitlab configuration file.
```
> cd /etc/gitlab
> vi /etc/gitlab/gitlab.rc (To update the configurations)
```
```
1) External URL: this will be your ec2 instance's fqdn. Change it to the IP address.

> external_url 'http://xx.xx.xx.xx'

2) Registry External URL: This is the URL where your docker registry will be available. In my case, i have provided "registry.srikanth.com". Change this parameter accordingly.

> registry_external_url 'https://registry.srikanth.com' 

3) Update Gitlab Rails registry path: This is the location where the registry files will be stored/uploaded. Just uncomment this line.

> gitlab_rails['registry_path'] = "/var/opt/gitlab/gitlab-rails/shared/registry"

4) Uncomment "registry['enable'] = true" line in the same config file. This will enable registry service on gitlab.

5) Uncomment "registry_nginx['enable'] = false" and change it to true.

> registry_nginx['enable'] = true

6) Add the SSL certificates locations. These will not show up in the default config. We just need to add them. And make sure the paths are correct.

> registry_nginx['ssl_certificate'] = "/etc/gitlab/trusted-certs/registry.srikanth.com.crt"
> registry_nginx['ssl_certificate_key'] = "/etc/gitlab/trusted-certs/registry.srikanth.com.key"

7) Uncomment gitlab_rails[lfs_enabled'] = true in the same config file.
> gitlab_rails['lfs_enabled'] = true
```

### Step 8: Update the /etc/hosts file to add a DNS entry.
```
> sudo vi /etc/hosts

Add :

xx.xx.xx.xx  ec2-xx-xx-xx-xx.compute-1.amazonaws.com  registry.srikanth.com

> Make sure you are able to ping the URL.
> ping registry.srikanth.com

PING ec2-xx-xx-xx-xx.compute-1.amazonaws.com (xx.xx.xx.xx) 56(84) bytes of data.
64 bytes from ec2-xx-xx-xx-xx.compute-1.amazonaws.com (xx.xx.xx.xx): icmp_seq=1 ttl=63 time=0.705 ms
64 bytes from ec2-xx-xx-xx-xx.compute-1.amazonaws.com (xx.xx.xx.xx): icmp_seq=2 ttl=63 time=0.553 ms
64 bytes from ec2-xx-xx-xx-xx.compute-1.amazonaws.com (xx.xx.xx.xx): icmp_seq=3 ttl=63 time=0.399 ms
^C
--- ec2-xx-xx-xx-xx.compute-1.amazonaws.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.399/0.552/0.705/0.126 ms
```

### Step 9: Update the gitlab with new configuration settings/file.
```
> sudo gitlab-ctl reconfigure
```

### Step 10: Access the gitlab UI page now.
#### Follow the step 3 to login to UI.
#### It will land in the "Projects" page.
![Alt text](Images/projects.png?raw=true "Projects")
#### Click on "Create a project".
```
> Provide a project name
> And click on "Create Project" name.
```
![Alt text](Images/cr_prj.png?raw=true "Create a project")
![Alt text](Images/cr_prj_res.png?raw=true "Projects page.")

```
If you see on the left side menu, and if you hover the mouse on "Packages & Registries" you will see the "Container Registry" link will be enabled.
```
![Alt text](Images/registry.png?raw=true "Projects page.")

```
Once you click on the "Container registry" you will see the below page.
```

![Alt text](Images/container_reg.png?raw=true "Projects page.")

### Step 10: Setup the docker host to use the container registry that was created.

`NOTE: In this case, the docker will be running on the same EC2 instance. You can run this in another host as well.`

```
1) 
> sudo apt update
```

```
2) Install some pre-reqs

> sudo apt install apt-transport-https ca-certificates curl software-properties-common
```

```
3) Add GPG key

> curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

```
4) Add docker repo

> sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
```

```
5) Update again

> sudo apt update
```

```
6) Install docker

> sudo apt install docker-ce
```

```
7) Add user to the docker group

> sudo usermod -aG docker ${USER}
> sudo usermod -aG docker ubuntu
```
```
8) Simple test:
> docker
```

### Step 11: Copy the ssl certs that were created above to docker's certs.d folder as below.

```
1) Create a folder with "certs.d/registry.srikanth.com" under /etc/docker folder

> sudo mkdir -p /etc/docker/certs.d/registry.srikanth.com
> sudo chown -R 755 /etc/docker/certs.d/
```

```
2) Copy the ssl cert file to the "certs.d/registry.srikanth.com" folder.

`Note:` In my case, docker host and gitlab hosts both are same. Otherwise you need to copy the ssl certs from the gitlab host to docker host.

> cd /etc/docker/certs.d/
> sudo cp -r -p /etc/gitlab/trusted-certs/registry.srikanth.com.crt .
> sudo cp -r -p registry.srikanth.com.crt ca.crt
> sudo chown -R 755 /etc/docker/certs.d/

> sudo cp -r -p registry.srikanth.com.crt /usr/local/share/ca-certificates/
> sudo update-ca-certificates
> sudo service docker restart
```

```
3) Reload the docker config

> sudo service docker reload
```

```
4) Connect to gitlab registry from docker.

> docker login registry.srikanth.com

It will prompt you for the username and password for the gitlab. Provide the same username and password that is used to login to the github UI.

Ouput:
======

sudo docker login registry.srikanth.com
Username: root
Password:
WARNING! Your password will be stored unencrypted in /home/ubuntu/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

```

### Step 12: Installing and setting up Gitlab Runner.
#### We will also setup the Gitlab runner on the same EC2 instance.

```
> cd
> mkdir gitlab-runner
> cd gitlab-runner
```
```
> curl -LJO https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb
> sudo dpkg -i gitlab-runner_amd64.deb
> sudo gitlab-runner status
```

```
Note: Get the gitlab coordinator url and token from the gitlab CI/CD page.
-> Login to Gitlab UI, go to CI/CD settings page, click on "Expand" button under "Runners" section
-> In this page, you can see URL for runner setup and token for registration. Copy those values.
```
![Alt text](Images/cicd.png?raw=true "CI/CD settings")
![Alt text](Images/runner.png?raw=true "Runners")

```
> sudo gitlab-runner register

> Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
**** http://xx.xx.xx.xx/
> Please enter the gitlab-ci token for this runner:
**** maskedtempasdklsadnqwencascn
> Please enter the gitlab-ci description for this runner:
**** [ip-xx-xx-xx-xx]: my-gitlab-runner
> Please enter the gitlab-ci tags for this runner (comma separated):
**** my-gitlab-runner, srikanth
> Registering runner... succeeded                     runner=a1uJroGA
> Please enter the executor: docker-ssh+machine, custom, parallels, shell, virtualbox, docker+machine, docker, docker-ssh, ssh, kubernetes:
**** docker
> Please enter the default Docker image (e.g. ruby:2.6):
**** alpine:latest
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
```

#### Next copy the same cert file that was created above using openssl in to gitlab-runner config directory.
```
> cd /etc/gitlab-runner/
> mkdir -p config/certs
> sudo cp -r -p /etc/gitlab/trusted-certs/registry.srikanth.com.crt .
> sudo cp -r -p registry.srikanth.com.crt ca.crt
> sudo rm registry.srikanth.com.crt
> mv ca.crt config/certs/

> pwd
/etc/gitlab-runner

> ls -ltra
total 16
drwxr-xr-x 96 root root 4096 Aug  5 02:32 ..
-rw-------  1 root root  530 Aug  5 02:34 config.toml
drwxr-xr-x  3 root root 4096 Aug  5 05:01 config
drwx------  3 root root 4096 Aug  5 05:03 .
```

```
#### Update the config.toml file.
> vi config.toml

> check for "volumes" under [runners.docker] section, and update with the location of the ca.crt file that we created.

> Map "/etc/gitlab-runner/config/certs/ca.crt" on the local gitlab runner host to docker container path.
 
> volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock", "/etc/gitlab-runner/config/certs/ca.crt:/etc/gitlab-runner/config/certs/ca.crt"]

```
#### Restart the gitlab runner service.
```
> sudo gitlab-runner status
> sudo gitlab-runner stop
> sudo gitlab-runner start
> sudo gitlab-runner status
```

```
Finally, you will see the above runner activated in the "Runners" page under CI/CD settings as below.
```
![Alt text](Images/runner-result.png?raw=true "Runners Result")

### Step 13: Running the jobs which are untagged.
```
Note: By default, if the jobs are untagged, they will not run, You need to update a setting in the "Runner" as below.
```
```
-> Click on the edit button, next to Runner as shown in the below screen.
-> Select the check box next to "Run Untagged jobs" setting as shown in the below image.
```
![Alt text](Images/runner-edit.png?raw=true "Runners edit")

### Step 14: Sample .gitlab-ci.yml file that works with the above configuration.
```
image: docker:18-git

variables:

services:
  - docker:18-dind

before_script:
  # Add this entry in the /etc/hosts file. This is needed for name resolution, otherwise it will use the "nameserver" ip address defined in the /etc/resolv.conf file and it will fail.
  # This entry needs to be the same entry that you added in your docker host's /etc/hosts/ file.
  # You can check step 8: in this https://github.com/srikanth311/Enabling-Container-Registry-In-Gitlab
  - echo '100.25.45.171	ec2-100-25-45-171.compute-1.amazonaws.com	registry.srikanth.com' >> /etc/hosts
  - cat /etc/hosts
  # In your gutlab-runner's "config.toml" file, make sure you map the voulmes from the docker host to docker service container.
  # On my gitlab runner host, I have this.
  # cat /etc/gitlab-runner/config.toml | grep volumes
  #    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock", "/etc/gitlab-runner/config/certs/ca.crt:/etc/gitlab-runner/config/certs/ca.crt"]
  # On my gitlab runner host, i have my ca.crt file in this location: /etc/gitlab-runner/config/certs/ca.crt
  # From this location, i am copying it to "/etc/gitlab-runner/config/certs/ca.crt" on the docker service container.
  # The ls command just checks if the directory exists in the docker container.
  - ls /etc/gitlab-runner
  # The below "cp" command copies the file from "/etc/gitlab-runner/config/certs/ca.crt" to "/usr/local/share/ca-certificates/ca.crt". This "/usr/local/share/ca-certificates/ca.crt" location
  # is the default location to store the certs.
  - cp /etc/gitlab-runner/config/certs/ca.crt /usr/local/share/ca-certificates/ca.crt
  # Checking again if the file gets copied successfully or not.
  - ls /usr/local/share/ca-certificates/ca.crt

stages:
  - build
  - deploy

build:
  stage: build
  services:
    - name: docker:dind
  script:
    - update-ca-certificates
    - echo "Certificates are updated."
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t registry.srikanth.com/root/simple-node-app .
    - docker push registry.srikanth.com/root/simple-node-app

``` 










