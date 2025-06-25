---
title: Web Server - Docker
categories: [ "Docker", "Apache2", "PHP" ]
image:
  path: ftp_docker.png
  alt: Apache2/Docker preview
layout: post
media_subpath: /assets/posts/2025-06-25-apache2-docker
tags: [ "Apache", "PHP", "Docker", "Linux", "DevOps", "Sysadmin", "Containers" ]
---

<!-->
## Objective

The goal here is to configure a web server (`Apache` in this case) inside a Docker container. This will have a login form which requires some user credentials which will be linked to a MySQL database. However, this integration will be done in a different post.
## Installation

In order to do start configuring our web server for our desires and, at the same time, avoid some painful headaches, we will download the official [PHP image](https://hub.docker.com/_/php) from [DockerHub](https://hub.docker.com/). In my case, i'll pick the `PHP 8.2` version running on `Apache`. 

Let's start by pulling the correct image and giving our new container a custom name. Remember to run this on detached mode (`-d`) because it will help us remove the need of opening another terminal tab. Also, we need to map the corresponding port like we did in the previous post which will allow us to access from a computer in our local network:

```bash
# Getting the image from DockerHub (We can skip this step and running the docker run command direclty since the image pull can happen automatically)
docker pull php:8.2-apache

# Running the container
docker run -d -p 8080:80 --name <container name> <image>
```

**Note**: Like i did in my previous FTP post, I will once again provide a script with the configuration ready to use so you don't have to worry about the manual method. However, as I also previously stated, I highly encourage you to walk with me through the manual method to really understand what you are doing and not simply fire off an automated script.

Once we have our container up and running, we will execute a shell inside of it so we can take a better look at the configuration of the container (I will use `bash` here, since the container supports this interpreter by default). For this, we run the following:

```bash
docker exec -it <container name> <command-line interpreter>
```

When we get inside the container, we will notice that our current working directory is `/var/www/html`, which is the standard webroot when it comes to PHP combined with Apache. In here is where we would like to have our files that will display our customised login form. 

However, how can we know where our webpage is being hosted? Web servers usually run on port 80 by default, but since we don't have a command like `ifconfig` installed on this container to check the correct interface, we will use the following command to discover that:

```bash
docker inspect <container name>
```

[docker_inspect_output.png](docker_inspect_output.png)

Now that we know the IP address and port where the web server is running, it is time to navigate to it and see what it will display. Apache by default will display a page that says `It works!` in case we installed everything correctly.

![apache2_default.png](apache2_default.png)

As we can see, we can confirm that we successfully installed our web server and it is up and running. Next, we will explore some configurations that need to be done in order to get what we want. 

**Note**: Now that we've mapped the ports, we can easily access our web server from our host machine's browser at `http://localhost:8080`. However, learning to find the container's internal IP address with `docker inspect` is still a valuable skill. It's essential for debugging and for scenarios where containers need to communicate with each other directly on Docker's internal network.
## The How and Why

Before jumping in, we have to think about how web servers work and the intended use they have. One extremely important concept to have in mind is what `vhost` are. `Vhosts` are similar to subdomains, but with the key difference that they are served on the same server and has the same IP address, such that one IP address could be serving several websites. 

Web servers implement this, which make us capable of hosting different web applications inside our container, as long as we have the required technology. Since the idea of containers is to test very specific things, we won't be changing the technology it uses. I would recommend to use a Virtual Machine in such cases. Alternatively, in modern cloud-native design, this is often handled by running each application in its own container and placing a _reverse proxy_ (like Nginx or Traefik) in front to direct traffic to the correct container.

The reason I made this very concise explanation about `vhosts`, is that it will allow me to introduce some core directories within Apache. Those are the following: 

| Directory         | Purpose                                                                      |
| ----------------- | ---------------------------------------------------------------------------- |
| `sites-available` | A library of all your website configurations, both active and inactive.      |
| `sites-enabled`   | A list of symbolic links pointing to the sites that are currently turned on. |

**Note**: We can find `apache2` configuration files inside `/etc/apache2/` directory.

To understand these directories better, think about the following scenario. Imagine you are hosting 10 different websites within your web server but you need one to go down due to maintenance. I am pretty sure you would not like to remove the whole configuration file of that particular web application from the container. This is where our `sites` directories shine.

The naming convention used by them are pretty self-explanatory, but I still consider it something important to bear in mind. If our site that needs maintenance is up and running, its configuration file will be inside the `/etc/apache2/sites-enabled` directory. To stop it from running without the need of deleting the file, we can run the following command:

```bash
a2dissite <my-web.conf>
```

Great job! Now our web application is down and ready for us to perform the desired changes. Up until this point, we got our configuration file removed from the `sites-enabled` directory but at the same time we conserve it inside the `sites-available` directory. Once we finish our changes, we can host it back again with the following command (we will see that the configuration file will once again appear inside `sites-enabled`):

```bash
a2ensite <my-web.conf>
```

Now that we have explored some core concepts, let's look into one example of a configuration file. For this, we will just take the default configuration file we saw in the beginning as an example:

```bash
cat /etc/apache2/sites-available/000-default.conf | grep -v "#" | sed -r '/^\s*$/d'
```

```html
<VirtualHost *:80>
	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html
	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

As we can see, here we are dealing with a virtual host that is listening on port 80 using all of its interfaces (`*` is a wildcard for this). In case of having multiple interfaces (like having two NIC card), we can also manually set on which interface we want our web application to listen. If we have `eth0` at 192.64.16.1 and `eth1` at 192.64.16.2 and we set this to `<VirtualHost 192.64.16.1:80>`, the `eth1` interface will ignore requests to this specific vhost since it is not listening on that interface.

The `ServerAdmin` just displays the contact email (not real) that Apache displays in some error pages. `DocumentRoot` is the most important directive because it indicates the root folder for our website files. The directory the instruction points to will be the one Apache will use as a reference to find the files that make our web page.

`ErrorLog` and `CustomLog` are the instructions that indicate the paths to the error and access log files. If Apache runs into an internal problem, it will write a detailed message about what it went wrong and when to the `error.log` file. `access.log` records every request that comes into the website (useful for monitoring). In both cases, we see `${APACHE_LOG_DIR}` which is just a variable that points to Apache's main logging folder, which is `/var/log/apache2/` by default.

We can confirm the value of this variable by going checking `/etc/apache2/envvars`:

```bash
grep APACHE_LOG_DIR /etc/apache2/envvars

#Output
: ${APACHE_LOG_DIR:=/var/log/apache2$SUFFIX}
export APACHE_LOG_DIR
```
## Configuration

You might be wondering why we were able to see a page being displayed when navigating to our container host on port 80 if there are no files in our `webroot`. This is due to that what we are seeing is NOT actually a file, but a default action that Apache executes when it finds the webroot empty.

To partially fix this, we need to create our custom `index.html` (or `index.php`) file and reload the service. Reloading the service forces Apache to re-scan the `DocumentRoot` directory, where it will now see our new file and serve it instead of the default page.

```bash
service apache2 reload
```

Nevertheless, what i have said so far introduces one critical problem: What happens when we have multiple files in our webroot? How would the server know which one to display as the default page? This is why we use the `DirectoryIndex` instruction in our configuration file to fix our issue. With this rule, we can tell the web server to always display the page we choose as default:

```html
<VirtualHost *:80>
	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html
	
<!-- Setting default page -->
	DirectoryIndex index.php 
	
	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Ill create an `index.php` file to include all the elements at once on a single file. The final script looks like the following:

```php
<?php

if ($_SERVER["REQUEST_METHOD"] == "POST") {
    echo "<h1>Thank you for loggin in. We are processing your request...</h1>";
} else {
 
    ?>

    <!DOCTYPE html>
    <html>
    <head>
        <title>Login</title>
    </head>
    <body>
        <h2>Please Log In</h2>
        <p>Enter your credentials to continue.</p>
        <form action="" method="post">
            <label for="username">Username:</label><br>
            <input type="text" id="username" name="username"><br><br>
            <label for="password">Password:</label><br>
            <input type="password" id="password" name="password"><br><br>
            <input type="submit" value="Log In">
        </form>
    </body>
    </html>

    <?php
}
?>
```

Here, i simply created a small two-way logic for the login form. When sending POST requests to the server with the required parameters, i want it to display the message of `Thank you for...`. Otherwise (`GET`), just display the login form to the user. Summarising a bit, we can say this  form currently doesn't validate credentials and just displays a message.

In our final integration, we will modify the `POST` request logic to query a MySQL database. We will intentionally write this query in an unsafe way, creating the `SQL Injection` vulnerability that will form the core of our attack chain. Since I want to show everything separately and then put everything together, I will leave this as it is and create the `Dockerfile` for it, which will server as an automation method. 

In the next post, I'll configure the MySQL database and then integrate everything (FTP, Apache, MySQL) in one final post to explain the attack chain.
## DockerFile

Let's create our custom image:

```bash
FROM php:8.2-apache

RUN apt-get update && \
    useradd -m -s /bin/bash apache2 && \
    a2dissite 000-default.conf && \
    rm /etc/apache2/sites-available/000-default.conf && \
    rm /etc/apache2/ports.conf

COPY xycxz.conf /etc/apache2/sites-available/
COPY ports.conf /etc/apache2

COPY index.php /var/www/html/

RUN a2ensite xycxz.conf && \
    chown -R apache2:apache2 /var/www/html

EXPOSE 8080

USER apache2
```

Before moving on, let's explain what we are doing here since i perform some changes along the way. Because I don't want this container to be running as privileged user (such as `root`), I created a standard user of `apache2`. This introduces a couple of changes in order to make the image work properly.

**Note**: We won't set a user password this time here because it is not needed for the attack chain I want to demonstrate. Nevertheless, we should always set strong passwords for any user in our environment and make sure the credentials are not hardcoded.

First of all, we should know that the `RUN` and `COPY` instructions in Docker perform actions _during the build process_; so the flow we want to follow is using the `root` user for this kind of instructions to avoid any kind of denied security permissions when executing them (reason why we are switching users at the end of the build). However, this bring us another problem since (on Linux) only the `root` user is allowed to bind a program to a network port below `1024`, and the default Apache configuration is set to listen on port `80`.

This is what forces us to change the port configuration and expose a different one (8080 in this case). Luckily for us, those changes are very easy to do. We just have to make the configuration file of our site to listen on that port and also change `ports.conf` to match that port. We should have the following:
##### xycxz.conf

```html
<VirtualHost *:8080>
	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html
	DirectoryIndex index.php 
	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
##### ports.conf

```html
Listen 8080

<IfModule ssl_module>
	Listen 443
</IfModule>

<IfModule mod_gnutls.c>
	Listen 443
</IfModule>
```

**Note**: Notice that I made this user the owner of the webroot (`chown`) which simulates a realistic scenario, because companies usually have this type of service accounts to manage specific tasks. 

With our configuration and Dockerfile files ready, now we can automate the whole process.
## Automation

Here it comes a `Makefile` once again: 

```bash
# ===================================================================
# Makefile for the Dockerized Apache Server
# To use:
#   1. make setup  (Run this first time to create required files)
#   2. make        (Builds the image and runs the container)
# ===================================================================

# --- Configuration Variables ---
IMAGE_NAME := apache2:xycxz
CONTAINER_NAME := apache2-lab
APACHE2_USER := apache2
SERVER_PORT := 8080 # This one has to be above 1024!
HOST_PORT := 8080 # Change this to the desired port

HOST_IP ?= $(shell hostname -I | awk '{print $1}')

# --- Makefile Targets ---
.PHONY: all setup build run stop clean logs shell help

all: help

help:
    @echo "Usage: make <target>"
    @echo ""
    @echo "Targets:"
    @echo "setup    Create all necessary configuration files."
    @echo "build  Build the Docker image."
    @echo "run    (Default) Stop old container and run a new one."
    @echo "stop   Stop and remove the running container."
    @echo "clean  Stop container, remove image, and delete generated files."
    @echo "logs   Follow the logs of the running container."
    @echo "shell  Get an interactive shell inside the running container."

setup:
	@# Create the Dockerfile
	@echo "Creating Dockerfile..."
	@cat <<EOF > Dockerfile

FROM php:8.2-apache

RUN apt-get update && \
    useradd -m -s /bin/bash ${APACHE2_USER} && \
    a2dissite 000-default.conf && \
    rm /etc/apache2/sites-available/* && \
    rm /etc/apache2/ports.conf

COPY xycxz.conf /etc/apache2/sites-available/
COPY ports.conf /etc/apache2

COPY index.php /var/www/html/

RUN a2ensite xycxz.conf && \
	chown -R ${APACHE2_USER}:${APACHE2_USER} /var/www/html

EXPOSE ${SERVER_PORT}

USER ${APACHE2_USER}
EOF

	@# Create our site config file
	@echo "Creating xycxz.conf..."
	@cat <<EOF > xycxz.conf

<VirtualHost *:${SERVER_PORT}>
	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html
	DirectoryIndex index.php
	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
EOF

	@# Create ports.conf customised file
	@echo "Creating ports.conf..."
	@cat <<EOF > ports.conf

Listen ${SERVER_PORT}

<IfModule ssl_module>
	Listen 443
</IfModule>

<IfModule mod_gnutls.c>
	Listen 443
</IfModule>
EOF

	@# Create index.php file
        @echo "Creating index.php..."
        @cat <<EOF > index.php

<?php

if ($_SERVER["REQUEST_METHOD"] == "POST") {

    echo "<h1>Thank you for loggin in. We are processing your request...</h1>";

} else {

    ?>

    <!DOCTYPE html>
    <html>
    <head>
        <title>Login</title>
    </head>
    <body>
        <h2>Please Log In</h2>
        <p>Enter your credentials to continue.</p>

        <form action="" method="post">

            <label for="username">Username:</label><br>
            <input type="text" id="username" name="username"><br><br>

            <label for="password">Password:</label><br>
            <input type="password" id="password" name="password"><br><br>

            <input type="submit" value="Log In">

        </form>
    </body>
    </html>

    <?php

}
?>
EOF

# Building Docker Image
 build: setup
    
    @echo "Building Docker Image: ${IMAGE_NAME}"
    @docker build -t ${IMAGE_NAME} .

# Stops any old container and runs a new one
run: build

    @echo "Stopping and removing existing container: ${CONTAINER_NAME}..."
    -docker stop $(CONTAINER_NAME) >/dev/null 2>&1 || true
	-docker rm $(CONTAINER_NAME) >/dev/null 2>&1 || true
	@echo "Starting new container..."
	@docker run -d --name $(CONTAINER_NAME) -p ${HOST_PORT}:${SERVER_PORT} ${IMAGE_NAME} 
    @echo "Container '$(CONTAINER_NAME)' started with Apache2 service exposed on IP: $(HOST_IP)"

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
	-rm -f Dockerfile ports.conf index.php xycxz.conf

# View container's logs
logs:
	@docker logs -f $(CONTAINER_NAME)

# Get a shell inside the running container
shell:
	@docker exec -it $(CONTAINER_NAME) /bin/bash
```

**Note**: This time I included a help menu so you understand what each option is doing!