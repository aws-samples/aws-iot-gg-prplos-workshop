### Lab Zero: Setup a deveveloper EC2 instance

## Step 1 - Launch instance

1.1 - Login to the AWS Console and deploy an EC2 instance `c5.metal`  (96 vCPU / 192 MiB RAM) using Ubunty Server 22.04 LTS as the OS. Allocate 128 GiB Disk. Allow SSH (TCP 22), HTTPs (TCP 443) from anywhere in the Internet (0.0.0.0/0). Use an existing key or create a new one. Read this guide if you have never used EC2: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html


## Step 2 - Install Docker

2.1 - SSH into to the newly created EC2 instnace. Add Docker's official GPG plus install some additional software

```bash

sudo apt-get update
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo apt-get update
sudo apt-get install -y \
    ca-certificates \
    curl 
```

2.2 - Add the docker repository to Apt sources

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```

2.3 - Install Docker

```bash
sudo apt-get install -y \
    skopeo \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin 
```

## Step 3 - Test Docker

3.1. - Test Docker

```bash
sudo docker run hello-world
# Hello from Docker!
# This message shows that your installation appears to be working correctly.
```

## Step 4 - Install QEMU

4.1 - Install QEMU

```bash
sudo apt-get install -y \
    net-tools \
    qemu-system 

```