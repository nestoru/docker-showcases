Now that we have docker running let us buid a custom useful image and use it. This is your first step anyway after installing docker unless you are OK with storing your images in public or third parties' SaS.

To host a secure docker registry you can use the docker-registry image as is but you also need a reverse proxy like apache with mod_proxy. In this showcase we will build 'regproxy', an image containing apache service which is configured to reverse proxy requests to a docker-registry service. Both services (registry and proxy) will be dockerized. The regproxy image will be stored in the registry itself.

### Building the images
```
$ #ssh into the host that would host the registry:
$ ssh dockreg.sample.com 

$ #run the basic registry image in a container named dockreg mapping port 5000 from container to port 5000 in host
$ docker run -p 5000:5000 -d --name=dockreg registry

$ #test that your container is running the registry:
$ curl http://localhost:5000

$ #stop the container and remove the image
$ docker rm -f dockreg

$ #This registry image has two problems. First by default when it runs the container will store the tagged images locally which is not good (docker images must be lightweight so do not store data internally). Second as you can see is available in a non secure endpoint (http). We will correct that. While doing so we will learn a little bit more about docker. We will build a dockerfile to create the regproxy image and store it in the source control (assuming a subversion base path has already been created), that way we can recreate from scratch the image if needed. Note that docker does not eliminate the need to backup your data which is externally hosted as I mentioned.
$ sudo apt-get install -y subversion tree apache2-utils #some needed dependencies for this exercise
$ sudo mkdir /var/docker-registry #to store registry data (images and repos))
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
$ htpasswd ~/workspace/docker/images/registry/proxy/registry-htpasswd ${username} #add authentication for username to htpasswd file

$ #create self signed key, cert and intermediate CA files with validity 10 years (better to buy the cert!)
$ export domain=dockreg.sample.com
$ openssl genrsa -out ${domain}CA.key 2048
$ openssl req -x509 -new -nodes -key ${domain}CA.key -days 3650 -out ${domain}CA.crt
$ openssl genrsa -out ${domain}.key 2048
$ openssl req -new -key ${domain}.key -out ${domain}.csr
$ openssl x509 -req -in ${domain}.csr -CA ${domain}CA.crt -CAkey ${domain}CA.key -CAcreateserial -out ${domain}.crt -days 10000

$ #copy the CA cert locally (needs to be done in all hosts that want to access this registry. To avoid this buy the cert instead!). Note that the recommended official docker hack won't work --> sudo cp dockreg.sample.comCA.crt /etc/docker/certs.d/dockreg.sample.com/ca.crt
$ sudo mkdir /usr/local/share/ca-certificates/${domain}
$ sudo cp ${domain}CA.crt /usr/local/share/ca-certificates/${domain}/
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
$ docker run -v /var/docker-registry:/tmp/registry -d --name=dockreg --restart=always registry #run the registry container without exposing the insecure port, pointing to the mounted local docker-registry directory and making sure it restarts after reboot or if it crashes
$ docker run -v ~/workspace/docker/images/registry/proxy/registry-htpasswd:/etc/apache2/htpasswd/registry-htpasswd -p 443:443 -d --name=regproxy --restart=always --link dockreg:DOCKREG regproxy #run the proxy pointing to a local htpasswd file (to allow adding new users without having to rebuild the image) , exposing the secure port, linking it to the registry container and making sure it starts after reboot or if it crashes

$ # Hit https://dockreg.sample.com:8443/ from a browser. It should ask for your credentials and take you to a secure registry welcome message

$ #Troubleshooting:
$ docker logs proxy #In case of failure find out if the proxy run successfully. If not then remove,fix,build and run again.
$ docker ps #to confirm the image is running and get its container id
$ docker ps -a #to see all running and stopped instances including status
$ docker exec <container name or id> ls -alrt  /var/log/apache2/ #to see last logs from running container

$ #Wrap up:
$ docker tag -f <image name or id> dockreg.sample.com/regproxy:latest #If everything went fine it is time to tag the images and repositories directories
$ docker login https://dockreg.sample.com #you should be able to login. If you are not you missed something above
$ docker push  dockreg.sample.com/regproxy:latest #This will take a while as you are uploading your image to the secure registry
$ #access https://dockreg.sample.com/v1/search?q= from a browser and you should see your image listed there. Look into  /var/docker-registry/ in the host to confirm the data is there. We want to make sure everything will work after a restart so run 'sudo reboot'
$ docker ps #must show the two containers running and the registry url should still respond that is hosting regproxy image
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

```
There is some security risk here because we are trusting an ubuntu image plus a registry image which is not own by us. To go around this we can create the ubuntu image from scratch and a registry image out of it per the Dockerfile for registry (https://github.com/docker/docker-registry/blob/master/Dockerfile). This way you will end up with a secure registry that stores its own image.
