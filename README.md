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
> openssl req -newkey rsa:4096 -nodes -sha256 -keyout /etc/gitlab/trusted-certs/registry.srikanth.com.key -x509 -days 365 -out /etc/gitlab/trusted-certs/registry.srikanth.com.crt

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

```

```
3) Reload the docker config

> sudo service docker reload
```

```
4) Connect to gitlab registry from docker.

> docker login registry.srikanth.com

It will prompt you for the username and password for the gitlab. Provide them.

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













