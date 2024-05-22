# Deploying a Flask App on AWS

## Overview

In this guide, you’ll work through the process of

1. purchasing a domain name,
2. configuring that domain name to point to a server,
3. setting up a server, and
4. configuring HTTPS.

This process will take some time, make sure to budget at least 1 hour.

## Background Terms

- _Domain Registrar_ - A domain registrar is a company that has been granted permission by ICANN (the Internet Standards body) to lease domain names to an end user for a yearly fee. You can get 1 `.me` domain name for free through the Github Education Student Developer Pack. If you prefer `.com` or `.org`, this will require a yearly fee of around $15 for each year you wish the domain to be active.
- _Domain Name Service (DNS)_ - DNS is the phonebook of the internet. It allows you to tell a browser when a person visits google.com, where the server that runs google.com lives.
- _Elastic Compute 2 (EC2)_ - EC2 is Amazon Web Service’s (AWS) virtual server hosting service.
- _SystemD_ - SystemD is a modern service standard for servers that allows application servers to run as background servers so they automatically start when the server boots, and they can easily be restarted.
- _Target Group_ - AWS Target Groups are the intermediary between load balancers (which receive internet traffic) and EC2 instances which run the web application.
- _Elastic Load Balancer (ELB)_ - ELBs are AWS’s primary internet traffic receipt mechanism. They can receive traffic from external sources (users) and forward that traffic to any internal source.

## Accounts

This guide will make use of [Namecheap](https://namecheap.com) as the Domain Registrar and [Cloudflare](https://cloudflare.com) as the DNS provider. You are welcome to use others as desired.

## Setup

We’ll start at the bottom with the server, then work our way up to the domain.

### Server Setup

1. Sign up for an account on AWS using their [Free Tier](https://aws.amazon.com/free).

> Warning: The free tier lasts for one year (12 calendar months) after which payment is required for the services used.

1. Once you have logged into the Console, use the search bar at the top to search for EC2, then click the service.
1. Under Resources, you'll see a yellow button labeled `Launch Instance`. Go ahead and click it.
   1. Under Names and Tags, give your server a good name like `Web Server`.
      ![Name and tags](images/ec2/name-tags.png)
   1. Under Application and OS Images, change the image type to `Ubuntu` then pick the most recent LTS from the AMI dropdown.
      ![Name and tags](images/ec2/ami.png)
   1. Under Instance Type, leave `t2.micro` selected.
      ![Name and tags](images/ec2/instance-type.png)
   1. Under Key Pair, click `Create new key pair`. Enter a name like `Web Server`, but leave all other settings as their default. Ensure the downloaded file when you click create is stored somewhere safe. If you lose this, you lose access to your server.
      ![Name and tags](images/ec2/create-key-pair.png)
   1. Under Network Settings, use the default settings.
      ![Name and tags](images/ec2/network.png)
   1. Under Configure Storage, use the default settings.
      ![Name and tags](images/ec2/storage.png)
   1. Click Launch Instance to the right.
1. Click on the link to return to the instances screen and wait a few moments while your server boots.
1. Once your server has an ip address, attempt to connect to it with: `ssh -i "<Login Key>.pem" ubuntu@<ip address>` (ex: `ssh -i "Web Server.pem" ubuntu@54.87.51.78`). Note: you may receive a warning on your first server connection, asking if you trust the key, type `yes` then hit enter. If you receive a permissions warning strengthen the permissions with `chmod 0600 "<Login Key>.pem"`.
1. Once logged into the server, run security updates: `sudo apt-get update && sudo apt-get upgrade -y`
1. Now install python: `sudo apt-get install -y python3-pip`
1. Type `exit` to leave the server and return to your machine.

### Application Setup

Congratulations! You have a server running in the cloud, but it has nothing on it. Let's fix that.

1. Login to your server again: `ssh -i "<Login Key>.pem" ubuntu@<ip address>`.
1. Create a directory to house your web application `mkdir flask-app`.
1. Exit the server.
1. Download this repository, extract the zip, then `cd` into the extracted folder.
1. Let's copy the `flask_app.py` file to the folder you just made on the server. `scp -i "<Login Key>.pem" flask_app.py ubuntu@<ip address>:~/flask-app/`
1. Do the same with the `requirements.txt`: `scp -i "<Login Key>.pem" requirements.txt ubuntu@<ip address>:~/flask-app/`

> Note: As you move forward with your project, any changes you make to either file on your local machine will need to be copied to the server with the same commands. Ditto goes for any new files.

1. Finally, copy the `flask-app.service` files: `scp -i "<Login Key>.pem" flask-app.service ubuntu@<ip address>:~/`
1. Log back into the server, then `cd flask-app`. Run `ls` to see that the `flask_app.py` and `requirements.txt` were copied successfully. If you don't see the files, stop here and reattempt the above or seek help until they are present.
1. Let's setup flask and try the server.

```
sudo pip3 install --break-system-packages -r requirements.txt
python3 flask_app.py
```

You should see a message that the app is running.

1. Stop the app with `<crtl> + c` then `cd ..`. Run `ls` to see the service file you copied earlier. If you don't see the file, stop here and reattempt the above or seek help until it is present.
1. Let's install the service.

```
sudo cp flask-app.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable flask-app
sudo service flask-app start
```

1. Now if you run `sudo service flask-app status` you should be able to see the service running.
1. NIf you need to change the service you can stop it with `sudo service flask-app stop` or restart it with `sudo service flask-app restart`.

### Network

All right, our service is running but isn't accessible at all to users, which means its kinda pointless, let's fix that.

1. In the AWS Console, using the search bar, search for `Security Groups`.
1. There should be two security groups, a `default` and a `launch wizard`. We are not going to touch the `default` group.
1. Name the `launch wizard` group to `Web Server SG`.

1. Make another security group named `load-balancer-sg`.
1. Under Inbound Rules, add two rules.
