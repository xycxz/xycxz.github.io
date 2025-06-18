---
title: FTP - Docker
categories: [ "Docker", "FTP" ]
image:
  path: ftp_docker.png
  alt: FTP/Docker preview
layout: post
media_subpath: /assets/posts/2025-06-18-ftp-docker
tags: [ "FTP", "Docker", "Linux", "DevOps", "Sysadmin", "Containers" ]
---



## Objective

My goal is to create a realistically configured FTP server that will become the initial point of compromise in our upcoming chained exploit demonstration.
## Building the Foothold

To kick off and start our configuration journey today, we will begin by creating a Docker container with the required image for our lab. In my case, i will download the official Ubuntu image that can be found [here](https://hub.docker.com/_/ubuntu). I'll pick the `latest` version at the time of writing this post.

```bash
docker run ubuntu:latest
```

If this is your first time downloading a Docker image, you will probably get a message similar to the following:

```bash
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
69c262fc30fc: Pull complete 
Digest: sha256:b59d21599a2b151e23eea5f6602f4af4d7d31c4e236d22bf0b62b86d2e386b8f
Status: Downloaded newer image for ubuntu:latest
```

As the message states, the image could not be found locally and had to go over the `Registry` to be able to download it. However, what is this `Registry` exactly? This is where all the images are stored, which is in the official [Docker Hub](https://hub.docker.com/) page.

Let's do a quick analysis to understand what our command is doing when it does not find a Docker image locally on our computer. For this, i will be using a tool called [Wireshark](https://www.wireshark.org/), which will help us monitor the network and check for data packages.

Because i am using a clean Linux Ubuntu distro inside a Virtual Machine, i will run the `ip a` command that comes installed by default to check the correct interface of where my web traffic is moving. Once we have that, i will run the following command:

```bash
sudo wireshark
```

This will open up a GUI interface for us. Here, we will simply double click on the interface we identified before (`enp0s1` in my case). We will probably see a lot of packets going back and forth, but don't worry about this noise for now.

Let's run the following command on an image that does not exists on our computer locally and check what is happening on the network:

```bash
# Command
docker pull ubuntu

# Output

Using default tag: latest
latest: Pulling from library/ubuntu
d69d4d41cfe2: Pull complete 
Digest: sha256:8a1f59ffb675680d47db6337b49d22281a139e9d709335b492be023728e11715
Status: Downloaded newer image for ubuntu:latest
docker.io/library/ubuntu:latest # Here we have a clue of the site our image is getting downloaded from
```

![wireshark-dns.png](wireshark-dns.png)

I applied the `dns` filter in order to demonstrate the moment my computer asked for the IP address of the Docker's servers. Because communication with the servers uses `TLS` encryption for security reasons, i'll apply the `tls` filter to show the handshakes being made to external servers:

![wireshark-tls.png](wireshark-tls.png)

This is the final proof that shows the place of where Docker is getting the images when it does not find them locally. This is a trivial and short demonstration i wanted to put here just to insist on the wise advice of always `try to understand what you are doing and why it is working`. This is the key to master any area of the cybersecurity landscape.

We will continue on building our image. We can check that we have installed it successfully by running the following command:

```bash
# Command
docker images

#Output
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
ubuntu        latest    9d45648b4030   2 weeks ago    101MB
```

In order to be able to configure our container and install everything what we need in a more comfortable way (meaning on a shell inside the container), we have to run it in `interactive` mode and allocate a `pseudo-TTY`. We will need the following flags for that:

![interactive-shell.png](interactive-shell.png)

I'll also assign a custom name to it to keep ourselves organised:

```bash
docker run -it --name ftp-lab ubuntu bash 
```

![docker-flags.png](docker-flags.png)

We find ourselves inside the container! It is very important to remark that containers are not prepared to work in the same way as an OS would work normally. We can prove that when trying to run commands like `ip` or `ifconfig`. 

This happens due to that a container is gonna be likely used for `something very specific` and it would not make any sense to install a bunch of tools we won't be using. Instead, we will be `choosing exactly what we need` and do it in a manual way so we can understand things better.

**Note**: I will be leaving a `Makefile` script that perform the same actions at the end of the post. You can use them to automate the whole process. However, i highly recommend going through the manual process first to understand the concepts.

Before starting, i will point out that this is NOT the ideal way of starting containers. In the best scenario, we would like to have our personalised images already configured so we can skip this whole process. However, since this is our first time, we have to build it from scratch.

**Note**: Do not forget to run `apt upgrade && apt update` inside the container as soon as you create it. This will allow us to install the tools we need.
## Getting Started with FTP

One of the most used FTP servers on Linux-based distros is [vsFTPd](https://security.appspot.com/vsftpd.html). We can install this with the following command:

```bash
apt-get install -y vsftpd
```

We will be prompted twice to choose our geographical area. Of course, in a real scenario we would have to put the corresponding data. However, since this is just a demonstration, we will choose whatever we want.

The configuration file can be found in `/etc/vsftpd.conf`. Let's take a look at the different options we can find here and give a small description of what they are for:

```bash
cat /etc/vsftpd.conf | grep -v "#" # We exclude "#" to skip comments
```

| **Setting**            | Value Example                          | **Description**                                                                                                                                                                                                                                              |
| ---------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `listen`               | NO                                     | When `YES`, `vsftpd` runs in standalone mode, listening directly on a port. If `NO`, it expects to be launched by another service (like `inetd` or `systemd` socket activation).                                                                             |
| `listen_ipv6`          | YES                                    | Enables `vsftpd` to listen for incoming connections on IPv6 addresses.                                                                                                                                                                                       |
| `anonymous_enable`     | NO                                     | Enables/Disable anonymous access.                                                                                                                                                                                                                            |
| `local_enable`         | YES                                    | Allows/Denies local users to log in via FTP.                                                                                                                                                                                                                 |
| `dirmessage_enable`    | YES                                    | Users will/won't see the contents of a `.message` file (if present) when they enter a directory. Useful for providing welcome messages or notices.                                                                                                           |
| `use_localtime`        | YES                                    | If `YES`, all directory listings will show timestamps in the server's local time zone. If `NO`, timestamps will be in GMT/UTC.                                                                                                                               |
| `xferlog_enable`       | YES                                    | Enables/Disables logging of all FTP uploads and downloads to `/var/log/vsftpd.log` (or a custom path). Essential for auditing.                                                                                                                               |
| `connect_from_port_20` | YES                                    | `vsftpd` will/won't use port 20 for active mode data connections. This is a security measure and is often required by FTP clients for active mode to work.                                                                                                   |
| `secure_chroot_dir`    | /var/run/vsftpd/empty                  | Specifies an empty directory that `vsftpd` will "chroot" into (change its perceived root directory to) _before_ processing requests. This is a strong security measure, preventing `vsftpd` from accidentally breaking out of the user's isolated directory. |
| `pam_service_name`     | vsftpd                                 | Specifies the PAM (Pluggable Authentication Modules) service name that `vsftpd` will use for authentication.                                                                                                                                                 |
| `rsa_cert_file`        | /etc/ssl/certs/ssl-cert-snakeoil.pem   | Specifies the path to the RSA certificate file for SSL/TLS (FTPS) encryption. Used when `ssl_enable` is `YES`.                                                                                                                                               |
| `rsa_private_key_file` | /etc/ssl/private/ssl-cert-snakeoil.key | Specifies the path to the RSA private key file corresponding to the certificate. Used when `ssl_enable` is `YES`.                                                                                                                                            |
| `ssl_enable`           | NO                                     | Enables/Disables SSL/TLS encryption. **For any production or sensitive use, this should be `YES` and proper certificates should be used.** If `NO`, connections are unencrypted (plain FTP).                                                                 |

These are default settings that comes with `vsftpd` installation. Of course, there are other optional settings that can be found [here](https://vsftpd.beasts.org/vsftpd_conf.html). Since there are a lot of them and it would take a long time to explain all of them, we are gonna tackle the ones that would suppose a critical vulnerability in the real world:

| **Setting**               | Value              | **Description**                                                                                                 |
| ------------------------- | ------------------ | --------------------------------------------------------------------------------------------------------------- |
| `anonymous_enable`        | YES                | Allows anonymous login                                                                                          |
| `anon_upload_enable`      | YES                | Allows anonymous user to upload files                                                                           |
| `anon_mkdir_write_enable` | YES                | Allows anonymous user to create directories                                                                     |
| `no_anon_password`        | YES                | Do not ask anonymous for password                                                                               |
| `anon_root`               | /home/username/ftp | Directory for anonymous.                                                                                        |
| `write_enable`            | YES                | Allows anonymous user to perform write operations on the server (if it is combined with `anonymous_enable=YES`) |

However, these are definitely NOT common to find, specially in the external network. This is why i don't see a point on configuring this wrongly on purpose. The only scenario i can think of that is possible to find those is on a forgotten FTP server, or one that it is running on a legacy system.

In a more realistic scenario, an FTP security flaw would likely be part of a chain of exploits. For example, imagine the FTP service is linked with a web server like `Nginx`. In that case, we might be able to write files inside the webroot of the web application, which could potentially lead to Remote Code Execution (RCE).

We will first create our own customised Docker image and configure it to run FTP on a Ubuntu server. The idea here is learning how to do this and be able to expose it inside our local network. Later we will also explore commands to use in order to move within the service. Let's see how this is done.
## Docker Container Configuration

Our `Dockerfile` will look like the following:

```bash
FROM ubuntu:22.04

ENV PASV_ADDRESS=127.0.0.1
ENV PASV_MIN_PORT=49200
ENV PASV_MAX_PORT=49210

RUN apt-get update && \
    apt-get install vsftpd -y && \
    useradd -m xycxz && \
    mkdir -p /var/run/vsftpd/empty && \
    mkdir -p /etc/vsftpd && \
    rm /etc/vsftpd.conf

RUN --mount=type=secret,id=xycxz_pwd \
    echo "xycxz:$(cat /run/secrets/xycxz_pwd)" | chpasswd

COPY vsftpd.conf /etc/vsftpd/vsftpd.conf.template

COPY passive.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/passive.sh

ENTRYPOINT ["/usr/local/bin/passive.sh"]

EXPOSE 20 21 49200-49210

CMD ["/usr/sbin/vsftpd", "/etc/vsftpd/vsftpd.conf"]
```

Let's explain this in a more detailed way. We start by importing the official Ubuntu image (we will use version 22.04 in our case):

```bash
FROM ubuntu:22.04
```

Next, we will define some environmental variables that will have some default values in case the user does not supply any. These will be linked to the script `passive.sh`, which we will see soon.

Allow me to do an important clarification on this point. The `PASV_ADDRESS` is the address the server `tells the external client to connect back to` when performing passive connections. If it tells the client to connect to `localhost`, the client will try to connect to itself, not to the server. 

Let's imagine we successfully connect to FTP through port 21. If we try to run a command such as `ls`, the `vsftpd` server will prepare for a `passive connection`. It looks at its configuration (`vsftpd.conf`), sees `PASV_ADDRESS=localhost`, and sends a message back to the client saying: "Okay, to get the file list, please open a new connection to `localhost` on port `21100` (in this case)."

Our FTP client that we are using in our laptop will receive the instruction being sent by the server. Because the localhost always makes reference to the machine we are currently on, our machine (client) will try to connect to itself on port 21100. This connection will eventually fail because we are trying to connect to an FTP service that is not running in our computer, but on a remote host.

So, to be able to connect to the actual server hosting the FTP server, we need the address of that server host. This is why this value needs to be dynamic, since in our tests we will be finding different types of IP addresses. 

Lastly, when we talk about the ports used by passive connections, it's important to say that they have to be `higher than 10000`, be exposed, and don't have any type of mismatch when running the docker container (`docker run`) and setting up the `vsftpd` configuration file.

```bash
ENV PASV_ADDRESS=127.0.0.1
ENV PASV_MIN_PORT=21100
ENV PASV_MAX_PORT=21110
```

After this, we run the commands that we need in order to install the service and make it function properly. Apart from this, we will be also creating a user to be able to connect to the FTP service since the user root is disabled in FTP by default.

```bash
RUN apt-get update && \
    apt-get install vsftpd -y && \
    useradd -m xycxz && \
    mkdir -p /var/run/vsftpd/empty && \
    mkdir -p /etc/vsftpd && \
    rm /etc/vsftpd.conf

RUN --mount=type=secret,id=xycxz_pwd \
    echo "xycxz:$(cat /run/secrets/xycxz_pwd)" | chpasswd

COPY vsftpd.conf /etc/vsftpd/vsftpd.conf.template
```

Here, we create a folder named `empty` because we will pretty much use a default configuration file and, if we recall, `secure_chroot_dir` points to that directory which is NOT created automatically in our case. We also create a `vsftpd` folder just to keep ourselves organised and copy a template file and our desired configuration file inside of it. The template file is used so it does not modifies the configuration file every time we start a container, because of what our `passive.sh` script is doing.

However, the most interesting part of this set up is how we managed the user password in order to not hardcode the credentials on this file. It is important to clarify that we are doing this in the `build` phase of the container, which means that we need to pass some special arguments to our `docker build` command. We won't be using the basic `--build-arg` method since the value of the argument is `recorded and easily visible` in the image's history and metadata. We can easily see this by running `docker history <IMAGE>`.

The superiority of `RUN --mount=type=secret` over `--build-arg` and `ENV` comes down to one critical concept:

Instead, we will use a different method which is using `Docker Build Secrets`. A "secret" is a file that we temporarily make available to a `single RUN command` during the build.  It is the only method that leaves absolutely no trace of the secret in the final Docker image or its history. Both `ENV` and `--build-arg` leave behind artifacts that can be discovered by anyone who gets a copy of your image. 

To make this work, first we need to create a file with out password:

```bash
echo "<password>" > ftp_password.txt
```

**Note**: The `ftp_password.txt` file needs to be in the SAME place of our Dockerfile archive.

Then, we have to use `RUN --mount=type=secret,id=xycxz_pwd`, which will tell Docker to mount a secret (we can choose any ID for it). Inside the `RUN` command's environment, the secret is available `as a file`. We use `$(cat /run/secrets/xycxz_pwd)` to read the content of the secret file (which is our password) and use it with `chpasswd` to set it up.

```bash
COPY passive.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/passive.sh

ENTRYPOINT ["/usr/local/bin/passive.sh"]
```

The previous lines just get our script ready to run once the blueprint of our image is finished. The last line simply executes the script. 

**Note**: Remember that every command we pass after `RUN` here will be used to create the image. This also applies when we define the `ENV` variables.

```bash
EXPOSE 20 21 49200-49210

CMD ["/usr/sbin/vsftpd", "/etc/vsftpd/vsftpd.conf"]
```

The remaining lines are used to expose the ports that FTP uses to be able to connect to it (and use passive communication), not only with the host machine, but with different machines in the local network. The final `CMD` commands executes the FTP service.

Now, i will show the configuration/script files i used here:
##### vsftpd.conf

```bash
ftpd_banner=Welcome to xycxz awesome FTP Server!
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
```
##### passive.sh

```bash
#!/bin/sh

set -e # Used any command fails to stop the entire script immediately

cp /etc/vsftpd/vsftpd.conf.template /etc/vsftpd/vsftpd.conf # Copies the template configuration file. We use this because every time we start a container the script will run and, if we don't specify this, the properties will be added every time

# The tee command adds the properties and we also make sure to use the ENV variables defined in our Dockerfile

tee -a /etc/vsftpd/vsftpd.conf <<EOF

# This will make it run in the foreground so the container does not stop
background=NO
# Enables passive mode, useful when the server is behind a firewall or NAT
pasv_enable=YES
# Docker already runs the container in a secure sandbox. We set this to don't have any weird issues
seccomp_sandbox=NO
# Best practice for compatibility with modern users
utf8_filesystem=YES
# Used to manage passive communications
pasv_address=${PASV_ADDRESS}
pasv_min_port=${PASV_MIN_PORT}
pasv_max_port=${PASV_MAX_PORT}
EOF

exec "$@" # This is used to indicate the script is done running so Dockerfile can run the last CMD command which initialises the FTP service
```

**Note**: This script is what makes our container `dynamic` since it requires environmental variables that are used in the creation of the blueprint the container will use. We can adapt this to our needs!
## Final Touch

We will try to simulate a real world scenario and mount a directory with different `helpdesk` tools that can be found in companies. Let's start by creating a folder and set the necessary permissions so our created user can write and read the contents of it:

```bash
mkdir helpdesk

# Docker containers have their own internal user IDs. To allow our user inside the container (UID 1000) to write to the "helpdesk" folder on the host machine, we need to make sure the folder on the host is also owned by UID 1000.
sudo chown -R 1000:1000 helpdesk 
```

Now that we have this set, we will run our container once again but including the corresponding flag to mount our directory. But before that, we need to build our image with the Dockerfile we just created. Because we are using a secret to set our new user password, we will run the following:

```bash
# This is required to run the build command
export DOCKER_BUILDKIT=1
```

```bash
docker build --secret id=<ID GIVEN>,src=<FILE WITH OUT PASSWORD> -t <IMAGE TAG> .
```

- `--secret`: This flag tells Docker we are providing a secret.
- `id=ftp_user_pass`: This `id` must match the `id` used in the `Dockerfile`'s `RUN --mount` command.
- `src=ftp_password.txt`: This points to the source file on your local machine that contains the password.

Now that our image is ready, we can run a container:

```bash
docker run -d --name <NAME> -p 21:21 -p 20:20 -p 49200-49210:49200-49210 -v /home/xycxz/helpdesk:/home/xycxz -e PASV_ADDRESS=<VM IP> -e PASV_MIN_PORT=49200 -e PASV_MAX_PORT=49210 <IMAGE>
```

I'll shortly explain what is this command doing:

- `-d`: Run in detached mode
- `--name`: Give it a custom name
- `-p`: Used for port mapping (21 for command, 20 for active data, 21100-21110 for passive data)
- `-e`: Used to pass the necessary environmental variables for our script
- `-v`: Used to mount a volume from our host inside the container. This way we can modify our directory on our host and it will apply the changes to the container as well.

Finally, we will put some files of tools that can be found in the context of a `helpdesk` team. We will run the following commands inside our `helpdesk` folder on the host VM for that:

```bash
git clone https://github.com/awesome-foss/awesome-sysadmin.git
git clone https://github.com/Sysinternals/sysinternals
mkdir -p documentation/bookstack_setup
cd documentation/bookstack_setup
git clone https://github.com/BookStackApp/BookStack.git
```

These will leave us with the following:

```bash
xycxz@ftp:/home/xycxz/# ls
awesome-sysadmin  documentation  sysinternals
```
## Automating the Lab with Make

We are finished setting up our FTP service. Now, we will proceed to show a scenario that implies some security flaws. However, before jumping into that, let me provide with the script necessary to automate our Docker container that we just created. We will use `Makefile` for this.
#### Definition

According to [IBM](https://www.ibm.com/docs/en/zvm/7.2.0?topic=make-finding-makefile), a "makefile is just a usual text file that (...) provides specific rules for remaking your targets." Its main purpose is to automate common tasks.

```bash
# ===================================================================
# Makefile for the Dockerized vsftpd Server
# To use:
#   1. make setup  (Run this first time to create files and set password)
#   2. make        (Builds the image and runs the container)
# ===================================================================

# --- Configuration Variables ---
IMAGE_NAME := ftp:xycxz
CONTAINER_NAME := ftp-lab
SECRET_FILE := xycxz_pwd.txt
SECRET_ID := xycxz_pwd
FTP_USER := xycxz
MOUNT_FOLDER := helpdesk
PASSIVE_PORTS := 49200-49210

# Auto-detects host IP
HOST_IP ?= $(shell hostname -I | awk '{print $1}')

# --- Makefile Targets ---
.PHONY: all setup build run stop clean logs shell

all: run

setup:
	@# Create the Dockerfile
	@echo "Creating Dockerfile..."
	@cat <<EOF > Dockerfile

FROM ubuntu:22.04

ENV PASV_ADDRESS=127.0.0.1
ENV PASV_MIN_PORT=$(firstword $(subst -, ,$(PASSIVE_PORTS)))
ENV PASV_MAX_PORT=$(lastword $(subst -, ,$(PASSIVE_PORTS)))

RUN apt-get update && \\
    apt-get install -y vsftpd && \\
    useradd -m $(FTP_USER) && \\
    mkdir -p /var/run/vsftpd/empty && \\
    mkdir -p /etc/vsftpd && \\
    rm -f /etc/vsftpd.conf && \\
    rm -rf /var/lib/apt/lists/*

RUN --mount=type=secret,id=$(SECRET_ID) \\
    echo "$(FTP_USER):\$$(cat /run/secrets/$(SECRET_ID))" | chpasswd

COPY vsftpd.conf /etc/vsftpd/vsftpd.conf.template

COPY passive.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/passive.sh

ENTRYPOINT ["/usr/local/bin/passive.sh"]

EXPOSE 20 21 $(PASSIVE_PORTS)

CMD ["/usr/sbin/vsftpd", "/etc/vsftpd/vsftpd.conf"]
EOF

	@# Create the vsftpd.conf template
	@echo "Creating vsftpd.conf template..."
	@cat <<EOF > vsftpd.conf

ftpd_banner=Welcome to $(FTP_USER)'s awesome FTP Server!
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
EOF

	@# Create the passive.sh script
	@echo "Creating passive.sh script..."
	@cat <<EOF > passive.sh
#!/bin/sh
set -e
cp /etc/vsftpd/vsftpd.conf.template /etc/vsftpd/vsftpd.conf
tee -a /etc/vsftpd/vsftpd.conf <<EOT
background=NO
pasv_enable=YES
seccomp_sandbox=NO
utf8_filesystem=YES
pasv_address=\$${PASV_ADDRESS}
pasv_min_port=\$${PASV_MIN_PORT}
pasv_max_port=\$${PASV_MAX_PORT}
EOT
exec "\$$@"
EOF

	@# Create the data directory for the volume mount
	@mkdir -p $(MOUNT_FOLDER)
	@echo "Created data directory: $(MOUNT_FOLDER)/"

	@# Prompt for the password and create the secret file
	@read -s -p "Enter password for FTP user '$(FTP_USER)': " FTP_PASSWORD; \\
	echo ""; \\
	echo "\$$FTP_PASSWORD" > $(SECRET_FILE); \\
	echo "Password saved to $(SECRET_FILE)."

# Builds the Docker image
build: setup
	@echo "Building Docker image: $(IMAGE_NAME)..."
	@export DOCKER_BUILDKIT=1 && \
	docker build --secret id=$(SECRET_ID),src=$(SECRET_FILE) -t $(IMAGE_NAME) .

# Stops any old container and runs a new one
run: build
	@echo "Stopping and removing existing container: $(CONTAINER_NAME)..."
	-docker stop $(CONTAINER_NAME) >/dev/null 2>&1 || true
	-docker rm $(CONTAINER_NAME) >/dev/null 2>&1 || true
	@echo "Starting new container..."
	@docker run -d --name $(CONTAINER_NAME) \
	  -p 21:21 \
	  -p 20:20 \
	  -p $(PASSIVE_PORTS):$(PASSIVE_PORTS) \
	  -v "$$(pwd)/$(MOUNT_FOLDER):/home/$(FTP_USER)" \
	  -e PASV_ADDRESS=$(HOST_IP) \
	  -e PASV_MIN_PORT=$(firstword $(subst -, ,$(PASSIVE_PORTS))) \
	  -e PASV_MAX_PORT=$(lastword $(subst -, ,$(PASSIVE_PORTS))) \
	  $(IMAGE_NAME)
	@echo "Container '$(CONTAINER_NAME)' started with FTP service exposed on IP: $(HOST_IP)"
	@echo "Log in with user '$(FTP_USER)' and the password you set."

# Stops and removes the running container
stop:
	@echo "Stopping and removing container: $(CONTAINER_NAME)..."
	-docker stop $(CONTAINER_NAME)
	-docker rm $(CONTAINER_NAME)

# Stops the container, removes the image, and deletes all generated files
clean: stop
	@echo "Removing Docker image: $(IMAGE_NAME)..."
	-docker rmi $(IMAGE_NAME)
	@echo "Removing generated files..."
	-rm -f Dockerfile vsftpd.conf passive.sh $(SECRET_FILE)
	-rm -rf $(MOUNT_FOLDER)

# View container's logs
logs:
	@docker logs -f $(CONTAINER_NAME)

# Get a shell inside the running container
shell:
	@docker exec -it $(CONTAINER_NAME) /bin/bash
```

We have the following options available:

- `make setup` to create all the files and set the password
- `make` (or `make run`) to build the image and start the container
- `make stop` to stop it
- `make clean` to erase everything and start over



