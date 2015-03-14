# docker-showcases
My journy with docker, unleashed. You may want to read https://github.com/wsargent/docker-cheat-sheet for a very good starting point. Be prepared to read a lot.

## What is docker?
It is a project that provides tools on top of Linux containers (LXC) to allow packaging of application environments in file images which can be run quickly in any host as a container. Images can be built my executing ad-hoc commands or by running them from inside a special file called 'dockerfile' using special docker commands although the RUN command literally allows to use any Plain Old Bash (POB) command.

## docker ubuntu quick start
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


