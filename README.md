# Akaunting with Docker
Build an akaunting Docker image and run with swarm / docker-compose

## Introduction
Getting Akaunting to work in docker took a little while. I could find very few resources online to address the issues I encountered. I hope this helps some other geeks out there.

These are the generalized files I used to get everything working. A docker image is built on an ubuntu image and apache. Ports 80 and 443 are exposed in the image and letsencrypt certbot is installed in case you need SSL.

The image building process downloads the latest release of Akaunting and installs all dependencies. The build process then copies a basic apache virtualhost config file and enables it as well a a php.ini file setup to work as required with Akaunting.

## Without SSL
If you don't need SSL all you need to do is configure the root akaunting.config apache virtualhost file in this repo with your domain and email address. Then run
> sudo docker build -t <your_image_tag_here> .

from repo root directory.
Then just
> sudo docker run -p 80:80 <your_image_tag_here> <give_a_name_to_your_container>

Congrats! Thats it. If you want to persist data and restart on failure just add your image tag to the docker-compose.yml file and 
> sudo docker-compose up -d

from repo root directory. docker-compose is setup to listen on 8002 for http and 8003 for https.

## With SSL
Now it gets ugly. But be patient and these instructions should get you squared away. We are going to use apache ProxyPass to setup a proxy on the host, but first we will use letsencrypt on the host. 
**BEFORE** firing up docker-compose:
> sudo apt install certbot python-certbot-apache

on host. Create a simple apache virtualhosts file (not gonna go over that here) to point to a blank html doc and enable site for port 80.
Then get a cert for the host:
> sudo certbot --apache -d www.yourdomain.com

Follow the setup wizard and select option to redirect http to https. 
**NOW** we can...
> sudo docker-compose up -d

from repo root directory.

Now the tricky part. First add back in the ProxyPass sections to the host apache virtualhost files. See examples in "host_apache_sites" folder for reference.

Now we need certs for our container. The docker-compose.yml files specifies volumes to persist these certs once created. Docker will not allow us to just ask for certs with apache like the host. We must accept dns challenges then get certs through apache.

### Step 1.
Get your running container name:
> sudo docker container ls

Then...
> sudo docker exec -it <your_container_name> /bin/bash

Now that we are executing commands inside the container we will validate certs using DNS.
> certbot -d yourdomain.com --manual --preferred-challenges dns certonly

You will be asked to create a txt record at your domain host. Create the record and wait a little while before continuing with the process. The domain must match exactly the domain validated on host.

### Step 2.
Now, while still in container a normal apache validation should work:
> sudo certbot --apache -d www.yourdomain.com

So what happens if you shut down the container? Well, the certs are safe in our volume, but you will have to run
> sudo certbot --apache -d www.yourdomain.com

from within the container and select option to reisntall existing certs.

Thats it! If this doesn't work for you or you know of a better way please let me know. 
