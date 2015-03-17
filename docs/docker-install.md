#Installation
At the time of this writting the original script is asking for unnecessary manual process. In any case the below plain old bash (POB) commands could be packaged in a simple recipe to install it remotely in any machine.

```
$ curl -sSL https://get.docker.com/ | sh 

$ #add current user to docker group:
$ sudo usermod -aG docker `logname`

$ #Login to the docker group:
$ newgrp docker

$ #Restart docker service:
$ sudo service docker restart

$ #Look at the logs for any isues:
$ sudo tail -100 /var/log/upstart/docker.log

$ #try docker by pulling an ubuntu image and running it in a container, then running the command 'hostname' in it to see as the response the default name of the image:
$ docker run ubuntu hostname
```
