# Docker Remote Lab

Tired of waiting for your Docker dependencies to download at your low internet local connection ? Use the Docker Remote Lab (DRL) architecture to benefit high speed and secure Docker developments.

## Pre-requisiste

- A VPS **on fiber** with Ubuntu 18.04 (e.g with [Scaleway](https://scaleway.com))
- Your local computer on Linux

## 1. Install - remote server

We are going to setup your VPS to install all stuff needed to dev at high speed from your local computer

- You must have here your **SSH keys configured** with your VPS
- Your VPS is disposable, you don't need to configure a user other than `root`

### 1a. Shared directory

This will allow you to directly work on your remote VPS.

We are going to use SSHFS that works through SSH :

```bash
apt install openssh-server sshfs -y
```

We want our work to lie in the `~/Dev` directory :

```bash
mkdir ~/Dev
```

### 1b. Add firewall

We don't want ports opened by Docker to be accessible from Internet, so let's configure a firewall to only allow port 22

```bash
apt install ufw -y
ufw allow 22 # only allow SSH traffic
ufw enable
```

### 1c. Install Docker

Let's [install the last version of Docker](https://docs.docker.com/engine/install/ubuntu/) and docker-compose :

```bash
apt-get update
apt-get install apt-transport-https ca-certificates curl gnupg lsb-release -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
    "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update
apt-get install docker-ce docker-ce-cli containerd.io docker-compose -y
```

Block external connections disabling Docker's iptables :

```bash
echo "{ \"iptables\": false }" > /etc/docker/daemon.json
```

### 1d. Configure the daemon

We want to expose our daemon so our local machine can connect to it.

Create a file `/etc/systemd/system/docker.service.d/options.conf` :

```bash
mkdir -p /etc/systemd/system/docker.service.d
nano /etc/systemd/system/docker.service.d/options.conf
```

Add the following config :

```conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H unix:// -H tcp://0.0.0.0:2375
```

Restart the daemon :

```bash
# Reload the systemd daemon.
sudo systemctl daemon-reload

# Restart Docker.
sudo systemctl restart docker
```

## 2. Install - local machine

You must have Docker installed on your machine before following the guide.

### 2a. Link to the remote directory

First you must install SSHFS :

```bash
apt install openssh-server sshfs -y
```

Let's say we're going to work in the `~/RemoteDev` directory. Let's mount it to our remote server :

```bash
mkdir -p ~/RemoteDev
sshfs -o IdentityFile=~/.ssh/id_rsa ~/RemoteDev root@<REMOTE_SERVER_IP>:/root/Dev # path must be absolute
```

> Unmount with `fusermount -u ~/RemoteDev`

### 2b. Establish SSH tunnels

In order to connect to the network of our VPS to access our Docker services, we have to establish a SSH tunnel :

```bash
ssh -4L 2376:127.0.0.1:2375 -N root@<REMOTE_SERVER_IP>
```

> You may want to save this line at the end of your `~/.bashrc` with the `-f` (background) option :
>
> ```bash
> ssh -4L 2376:127.0.0.1:2375 -fN root@<REMOTE_SERVER_IP>
> ```

### 2c. Use the remote VPS' Docker daemon

This is the key point of this architecture. Configure your local Docker client to point to your VPS remote daemon :

```bash
export DOCKER_HOST=tcp://localhost:2376
```

Test the daemon :

```bash
docker run hello-world:latest
```

> You may want to save this line at the end of your `~/.bashrc`

### 2d. Develop !

You can now develop on your remote server.

Remind yourself :

- Volumes must be **mounted from `/root/Dev/*`** as it is the shared folder in your remote VPS
- SSH-Forward ports used by your containers with commands similar to `ssh -4L 8080:172.17.0.1:8080 -fN root@<REMOTE_SERVER_IP>`
