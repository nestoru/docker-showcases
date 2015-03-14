# docker-showcases
This is my journey with docker, unleashed. Besides the official documentation there are two links that I found useful in learning the bit that I know today abouyt docker: https://github.com/wsargent/docker-cheat-sheet and https://docs.docker.com/articles/dockerfile_best-practices/.

## What is docker?
It is a project that provides tools on top of Linux containers (LXC) to allow packaging of application environments in file images which can be run quickly in any host as a container. Images can be built my executing ad-hoc commands or by running them from inside a special file called 'dockerfile' using special docker commands although the RUN command literally allows to use any Plain Old Bash (POB) command.

## Quick start with docker on Ubuntu
* let us install docker (these are plain old bash (POB) commands so a simple recipe to install it remotely in any machine should be piece of cake.
` curl -sSL https://get.docker.com/ | sh 
* add current user to docker group:
` sudo usermod -aG docker `logname`
* Login to the docker group 
` newgrp docker
* Restart docker
` sudo service docker restart
*  Look at the logs for any isues:
` tail -100 /var/log/upstart/docker.log
* Try docker by pulling an ubuntu image and running it in a container, then running the command 'hostname' in it to see as the response the default name of the image.
` docker run ubuntu hostname

## Showcase #1: Running your private secure registry
Now that we have docker running let us buid a custom useful image and use it. 

To host a secure docker registry you can use the docker-registry image as is but you also need a reverse proxy like apache with mod_proxy. In this showcase will build 'regproxy', an image containing apache service which is configured to reverse proxy requests to a docker-registry service. Both service will be dockerized. The regproxy image will be stored in the registry itself.

### Building the images
```
* ssh into dockreg.sample.com 
* Run the basic registry image in a container named docreg mapping port 5000 from container to port 5000 in host
<pre>
docker run -p 5000:5000 -d --name=dockreg registry
</pre>
* test your container is running the registry:
<pre>
curl http://localhost:5000
</pre>
* stop the container and remove the image
<pre>
docker rm -f dockreg
</pre>
* This registry image has two problems. First by default when it runs the container will store the tagged images locally which is not good as it will not persist. Second as you can see is publicly available in a non secure endpoint (http). We will correct that. While doing so we will learn a little bit more about docker. We will build a dockerfile to create the regproxy image and store it in the source control (assuming a subversion base path has already been created), that way we can recreate from scratch the image if needed.
<pre>
$ sudo apt-get install -y subversion tree apache2-utils #some needed dependencies for this exercise
$ sudo mkdir /var/docker-registry #to store data and db
$ sudo chown `logname`:`logname` /var/docker-registry
$ mkdir -p ~/workspace
$ cd workspace/
$ svn co http://subversion.sample.com/repos/docker/
$ cd workspace/
$ mkdir -p docker/images/registry/proxy
$ cd docker
$ svn add images/registry/proxy
$ svn ci -m "docker registry project first commit"
$ cd images/registry/proxy
$ echo ".svn" > .dockerignore #Add to this file anything not needed by the docker image
$ htpasswd registry-htpasswd ${username} #add authentication for username to htpasswd file
$ #create self signed key, cert and intermediate CA files with validity 10 years (better to buy the cert!)
$ export domain=dockreg.sample.com
$ openssl req -x509 -new -nodes -key devdockerCA.key -days 3650 -out devdockerCA.crt
$ openssl genrsa -out ${domain}CA.key 2048
$ openssl req -x509 -new -nodes -key ${domain}CA.key -days 3650 -out ${domain}CA.crt
$ openssl genrsa -out ${domain}.key 2048
$ openssl req -new -key ${domain}.key -out ${domain}.csr
$ openssl x509 -req -in ${domain}.csr -CA ${domain}CA.crt -CAkey ${domain}CA.key -CAcreateserial -out ${domain}.crt -days 10000
$ #copy the CA cert locally (needs to be done in all hosts that want to access this registry. To avoid this buy the cert instead!). Note that the recommended official docker hack won't work --> sudo cp dockreg.sample.comCA.crt /etc/docker/certs.d/dockreg.sample.com/ca.crt
$ sudo mkdir /usr/local/share/ca-certificates/${domain}
$ sudo cp ${domain}CA.crt /usr/local/share/ca-certificates/dockreg.sample.com/
$ sudo update-ca-certificates
$ cat ssl-proxy.conf #create virtual host apache configuration pointing to above created files 
<VirtualHost *:443>
        ServerName dockreg.sample.com 

        SSLEngine on
        SSLCertificateFile /etc/apache2/ssl/dockreg.sample.com.crt
        SSLCertificateKeyFile /etc/apache2/ssl/dockreg.sample.com.key

        Header set Host "dockreg.sample.com"
        RequestHeader set X-Forwarded-Proto "https"

        ProxyRequests     off
        ProxyPreserveHost on
        ProxyPass         / http://${DOCKREG_PORT_5000_TCP_ADDR}:5000/
        ProxyPassReverse  / http://${DOCKREG_PORT_5000_TCP_ADDR}:5000/

        ErrorLog ${APACHE_LOG_DIR}/dockreg.sample.com.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/dockreg.sample.com.log combined

        <Location />
                Order deny,allow
                Allow from all

                AuthName "Docker Registry Authentication"
                AuthType basic
                AuthUserFile "/etc/apache2/htpasswd/registry-htpasswd"
                Require valid-user
        </Location>

        # Allow ping and users to run unauthenticated.
        <Location /v1/_ping>
                Satisfy any
                Allow from all
        </Location>

</VirtualHost>
$ cat Dockerfile #create the Dockerfile
FROM ubuntu:14.04

RUN apt-get update

RUN apt-get -y install apache2

ADD ./registry-htpasswd /etc/apache2/htpasswd/
ADD ./dockreg.sample.com.* /etc/apache2/ssl/
ADD ./ssl-proxy.conf /etc/apache2/sites-available/
RUN rm -f /etc/apache2/sites-enabled/*
RUN ln -s /etc/apache2/sites-available/ssl-proxy.conf /etc/apache2/sites-enabled/ssl-proxy.conf
RUN mkdir /var/lock/apache2
RUN a2enmod ssl
RUN a2enmod headers
RUN a2enmod proxy
RUN a2enmod proxy_http
RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf

env APACHE_RUN_USER    www-data
env APACHE_RUN_GROUP   www-data
env APACHE_PID_FILE    /var/run/apache2.pid
env APACHE_RUN_DIR     /var/run/apache2
env APACHE_LOCK_DIR    /var/lock/apache2
env APACHE_LOG_DIR     /var/log/apache2
env LANG               C

CMD ["apache2", "-D", "FOREGROUND"]
$ docker stop regproxy dockreg #stop the containers in case they are running
$ docker rm -f regproxy dockreg #remove regproxy and dockreg containers in case they exist
$ docker build -t regproxy . #build the image, name it regproxy. The image id is provided upon completion
$ docker run -v /var/docker-registry:/tmp/registry -d --name=dockreg --restart=always registry #run the registry container without exposing the insecure port, pointing to the mounted local docker-registry directory and make sure it restarts after reboot or if it crashes
$ docker run -p 443:443 -d --name=regproxy --restart=always --link dockreg:DOCKREG regproxy #run the proxy exposing the secure port, linking it to the registry container and make sure it starts after reboot or if it crashes
$ # Hit https://dockreg.sample.com:8443/ from a browser. It should ask for your credentials and take you to your secured registry welcome message
$ docker logs proxy #In case of failure find out if the proxy run successfully. If not then remove,fix,build and run again.
$ docker ps #to confirm the image is running and get its container id
$ docker ps -a #to see all running and stopped instances including status
$ docker exec <container name or id> ls -alrt  /var/log/apache2/ #to see last logs from running container
$ docker tag -f <image name or id> dockreg.sample.com/regproxy:latest #If everything went fine it is time to tag the images and repositories directories
$ docker login https://dockreg.sample.com #you should be able to login. If you are not you missed something above
$ docker push  dockreg.sample.com/regproxy:latest #This will take a while as you are uploading your image to the secure registry
$ # Access https://dockreg.sample.com/v1/search?q= from a browser and you should see your image listed there.
$ # Look into  /var/docker-registry/ in the host and you will see 
$ #we want to make sure everything will work after a restart so run 'sudo reboot'
$ docker ps #must show the two containers running and the registry should still respond that is hosting regproxy image
$ tree ~/workspace/docker #This is the status of the docker projects so far which includes just building an internal secure registry
~/workspace/docker
└── images
    └── registry
        └── proxy
            ├── Dockerfile
            ├── dockreg.sample.comCA.crt
            ├── dockreg.sample.comCA.key
            ├── dockreg.sample.com.crt
            ├── dockreg.sample.com.csr
            ├── dockreg.sample.com.key
            ├── registry-htpasswd
            └── ssl-proxy.conf

</pre>
```
There is some security risk here because we are trusting an ubuntu image plus a registry image which is not own by us. To go around this we can create the ubuntu image from scratch and a registry image out of it per the Dockerfile for registry (https://github.com/docker/docker-registry/blob/master/Dockerfile). This way you will end up with a secure registry that stores its own image.







