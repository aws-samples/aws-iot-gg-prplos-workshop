### Lab Zero: Setup a deveveloper EC2 insrtance

## Step 1 - Launch instance

1.1 - Deploy EC2 instance `c5.metal`. Allow SSH (22), HTTPs (443) and a custom TCP port (1234) from the outside.

## Step 2 - Install Docker

2.1 - Add Docker's official GPG and install some tools

```bash

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
    qemu-system \

```