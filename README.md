# docker-showcases
This is my journey with docker, unleashed. Besides the official documentation there are two links that I found useful in learning the bit that I know today about docker: https://github.com/wsargent/docker-cheat-sheet and https://docs.docker.com/articles/dockerfile_best-practices/.

## What is docker?
It is a project that provides tools on top of Linux containers (LXC) to allow packaging of services in file images, storing them in repositories and running them as isolated containers in any host. Images can be built by executing ad-hoc commands or by running special docker commands from inside a special file called 'Dockerfile'. The RUN command allows to use any Plain Old Bash (POB) command.

## What should we care?
Images are a fraction of what a whole OS is so keeping services configuration is lighter than relying on storing a whole VM.

## [Quick start with docker on Ubuntu](docs/docker-install.md)

## [Showcase #1: Running your private secure registry](docs/docker-registry.md)







