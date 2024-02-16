# Creating an EC2 Instance with reverse proxy

* [Scenario](#scenario)

* [Create Instance from Scratch](#create-instance-from-scratch)

* [Creating instance from this repo](#creating-an-instance-from-this-repo)

## Scenario

I recently took on a client that had three different WordPress repositories that needed improvements and maintenance. I need a dev environment where I can safely test changes. I need to be able to conveniently demo these changes to the client. I want to minimize cost. I have 750 hours/month of free compute on AWS and a domain I have camped that I can lend to this endeavor. 

### Objective 

    I want a dev environment that will run in one instance on EC2 and support at least 3 applications mapped to different subdomains of a domain that I will supply.

## Create Instance from Scratch
### Prerequisites
*  AWS Developer account (I chose the 12 month free tier)
*  Domain name (I use route53, it doesn't need to be AWS)
### Steps
1. [Create an EC2 Instance](#create-an-ec2-instance)
    1. Create an EC2 instance
    1. Connect to instance with VS Code
    1. Baseline prep your instance
1. [Install Node](#install-node)
1. [Test Connectivity](#test-connectivity)
    1. Create something to serve
    1. Configure supporting libraries
    1. Update your EC2 instance security groups
    1. Server servers
1. [Install Docker](#install-docker)
    1. Install Docker
    1. Install lazydocker cli tool
1. [Set Up Reverse Proxy](#set-up-reverse-proxy)
    1. Configure subdomains in DNS manager to point to server
    1. Test
    1. Configure your workspace
    1. Run your servers
    1. Clean up your EC2 instance security group
### Create an EC2 Instance
1. Go to AWS console, create an EC2 instance, I picked freemium Ubuntu
    *  Max out free tier configuration parameters
    *  During this process, create a PEM key and save it off
    *  Your instance should be running
1. Connect to instance with SSH using PEM (I am going to do this through VS Code)
    *  Store the PEM somewhere, I also stored a backup of the PEM somewhere secure
    *  Change its permissions
        ```sh
            chmod 400 /path/to/your/key.pem
        ```
    *   Open VS code > Extensions and install Remote-SSH extension
    *   Edit or create an SSH configuration file to define your EC2 connection.
        > Press F1 to open the command palette, type **Remote-SSH: Open Configuration File**, and select it

        > You might be asked to choose a configuration file location; **~/.ssh/config** is a common choice.
        ```sh
            #Add this data to your config
            Host A-friendly-name
                HostName your-ec2-public-dns
                User ubuntu #specific to the Ununtu template
                IdentityFile /path/to/your/key.pem
                IdentitiesOnly yes #identity file authentication only
        ```
    *   Connect to Your EC2 Instance
        > Press F1 to open the command palette, type **Remote-SSH: Connect to Host...**, and press Enter.
            
        > Choose the friendly name you gave to your EC2 instance in the SSH configuration file.
    *  You should now have a VSCode instance connected to your EC2 instance
1. Baseline prep your instance
    ```sh
        # Update the Package Repository Cache
        sudo apt update

        # Upgrade Installed Packages
        sudo apt upgrade

        # Remove Unnecessary Packages
        sudo apt autoremove

        # Reboot the System
        sudo reboot
    ```
### Install Node
1. Install Node Version Manager (NVM) within your EC2 instance
    ```sh
        #install node version manager
        curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
    ```
1. Restart Your Shell
    ```sh
        #Open a new terminal or run this if using bash
        source ~/.bashrc
    ```
1. Install Node.js
    ```sh
        #install
        nvm install node
        #verify install
        node -v
    ```
### Test Connectivity
1. Create something to serve
    *   Create server/index.html
        ```html
            <h1>Hello Kitty!</h1> 
        ```
    *   Create server1/index.html
        ```html
            <h1>Hello Kitty!</h1> 
            <p>On server 1</p>
        ```
    *   Create server2/index.html
        ```html
            <h1>Hello Kitty!</h1> 
            <p>On server 2</p>
        ```
    *   Create server3/index.html
        ```html
            <h1>Hello Kitty!</h1> 
            <p>On server 3</p>
        ```
1. Configure supporting libraries
    *   Install authbind
        ```sh
            sudo apt -y install authbind
        ```
    *   Configure authbind ([reference](https://manpages.ubuntu.com/manpages/xenial/man1/authbind.1.html))
        ```sh
            # Create a file for port 80 to allow binding
            sudo touch /etc/authbind/byport/80

            # Change ownership of the file to the specified user
            sudo chown ubuntu /etc/authbind/byport/80

            # Set permissions to allow execution by the owner only
            sudo chmod 500 /etc/authbind/byport/80
        ```
1. Update your EC2 instance security groups
    *   Go to your EC2 Instance security groups
    *   Add rules for port 3001, 3002, and 3003
    *   You should already have one for port 80
1. Server servers
    *   Serve from `server`, then go to your EC2 instance public DNS, use HTTP, not HTTPS
        ```sh
            #This will serve index.html on port 80
                #Any port below 1024 is generally restricted
                #Authbind allows non sudo port 80 service
            authbind --deep npx serve -p 80
        ```
    *   Serve from `server1`, then go to your EC2 instance public DNS port 3001, use HTTP, not HTTPS
        ```sh
            npx serve -p 3001
        ```
    *   Serve from `server2`, then go to your EC2 instance public DNS port 3002, use HTTP, not HTTPS
        ```sh
            npx serve -p 3002
        ```
    *   Serve from `server3`, then go to your EC2 instance public DNS port 3003, use HTTP, not HTTPS
        ```sh
            npx serve -p 3003
        ```
### Install Docker
1.  Install docker, follow [docker installation instructions](https://docs.docker.com/engine/install/)
    *   Uninstall old versions
        ```sh
            #Ubuntu
            for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
        ```
    *   Set up Docker's apt repository
        ```sh
            # Add Docker's official GPG key:
            sudo apt-get update
            sudo apt-get install ca-certificates curl
            sudo install -m 0755 -d /etc/apt/keyrings
            sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
            sudo chmod a+r /etc/apt/keyrings/docker.asc

            # Add the repository to Apt sources:
            echo \
            "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
            $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
            sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
            sudo apt-get update
        ```
    *   Install the Docker packages
        ```sh
            #Install
            sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

            #start the service
            sudo service docker start

            #enable docker to start on boot
            sudo systemctl enable docker

            #add ubuntu as user
            sudo usermod -aG docker ubuntu

            #refresh session
            newgrp docker
            
            #verify install
            docker run hello-world
        ```
1. [Install lazydocker](https://github.com/jesseduffield/lazydocker?tab=readme-ov-file#binary-release-linuxosxwindows), this gives you a CLI/GUI interface instead of going straight command line
    > Install downloads binary to $HOME/.local/bin directory by default
    ```sh
        #install docker
        curl https://raw.githubusercontent.com/jesseduffield/lazydocker/master/scripts/install_update_linux.sh | bash

        #add $HOME/.local/bin to path
        export PATH=$PATH:$HOME/.local/bin

        #start lazydocker
        lazydocker
    ```
### Set Up Reverse Proxy
1. Go to your DNS manager (AWS route53), create a DNS record for each subdomain
    * For each subdomain, the route with be the public IPV4 address from the EC2 Instance. I am using `smoothkitties` for a domain.
    <p align="center">
        <img src="https://github.com/mchantosa/smooth-kitty/blob/main/static/images/myCat.jpg?raw=true" alt="My sphynx, Shiva"/>
    </p>
    
    *   Set smoothkitties to EC2 public facing IP

    *   Set site1.smoothkitties to EC2 public facing IP

    *   Set site2.smoothkitties to EC2 public facing IP

    *   Set site3.smoothkitties to EC2 public facing IP
1.  Test
    *   Start server on port 80. Smoothkitties.com, site1.smoothkitties.com, site2.smoothkitties.com, and site3.smoothkitties.com, should route to your EC2 instance server.
    *   Start server1 on port 3001. Smoothkitties.com:3001, site1.smoothkitties.com:3001, site2.smoothkitties.com:3001, and site3.smoothkitties.com:3001, should route to your EC2 instance server1.
1.  Configure your workspace
    *   Create a workspace directory and move your server directories into it. I renamed them site0, site1, site2, and site3
    *   Create docker-compose.yml in your workspace directory
        ```sh
            version: '3'  #file format version

            services:
            nginx:  #Nginx service
                image: nginx:alpine #image
                ports:
                - "80:80" #maps port 80 on the host to port 80 in the container
                volumes:
                - ./nginx.conf:/etc/nginx/nginx.conf:ro #mounts the local nginx.conf
                depends_on: #ensures nginx starts after web0, web1,...
                - web0
                - web1  
                - web2
                - web3
            
            web0:
                image: node:alpine
                command: sh -c 'npx -y serve -p 8000'
                volumes:  #Think container's hard drive
                - ./site0:/usr/app/site0  #mounts the local directory to the container
                working_dir: /usr/app/site0  #sets the working directory inside container
                ports:
                - "8000:80"

            web1:
                image: node:alpine
                command: sh -c 'npx -y serve -p 8001'
                volumes:  
                - ./site1:/usr/app/site1  
                working_dir: /usr/app/site1 
                ports:
                - "8001:80"

            web2:
                image: node:alpine
                command: sh -c 'npx -y serve -p 8002'
                volumes:
                - ./site2:/usr/app/site2
                working_dir: /usr/app/site2
                ports:
                - "8002:80"

            web3:
                image: node:alpine
                command: sh -c 'npx -y serve -p 8003'
                volumes:
                - ./site3:/usr/app/site3
                working_dir: /usr/app/site3
                ports:
                - "8003:80"
        ```
    *   Create nginx.conf in your workspace directory
        ```sh
            events {}

            http {

                server {
                    listen 80;
                    server_name smoothkitties.com;
                    location / {
                        proxy_pass http://web0:8000;
                    }
                }
                server {
                    listen 80;
                    server_name site1.smoothkitties.com;
                    location / {
                        proxy_pass http://web1:8001;
                    }
                }
                server {
                    listen 80;
                    server_name site2.smoothkitties.com;

                    location / {
                        proxy_pass http://web2:8002;
                    }
                }
                server {
                    listen 80;
                    server_name site3.smoothkitties.com;

                    location / {
                        proxy_pass http://web3:8003;
                    }
                }
            }
        ```
1.  Run your servers
    *   Run lazydocker
    *   From workspace, use a combination of lazydocker and the following to work out any kinks
        ```sh
            #Cleans up all the components that were deployed using the docker-compose.yml file in the current directory
            docker compose down

            #Start and run an entire Docker application defined in a docker-compose.yml file.
            docker compose up

            #use CTR + C to shut down servers or use lazydocker interface
        ``` 
1.  Clean up your EC2 instance security group
## Creating instance from this repo
... later



