# Using the Discord Hashcat Bot in a Container

You want to run this bot as a containerized work load, eh?  
There are a couple of things we need to take care of first. If you are a linux noob, this may be a hassle, but I tried to find the most painless way.

## Docker Setup

We need to install docker then set it up for rootless operation...

### Docker Install

#### Add Official GPG Key

```bash
sudo apt update -y 
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

#### Add Repository to Apt Sources

First we make sure there is no previous docker installations leftover. Apt might lie to you.

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y

sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Now you should be able to run the hello-world

`sudo docker run hello-world`

### Docker Rootless

`sudo apt install -y dbus-user-session`

Lets stop docker for a second

`sudo systemctl disable --now docker.service docker.socket`

If you installed docker from the instructions [above]() then docker has a rootless setup shell script we can use.  
Use `--force`, it will complain even though we shut down the service.

`dockerd-rootless-setuptool.sh install --force`

Now this should suffice to run the hello-world container.

`docker run hello-world`

Turn docker back on.

`sudo systemctl enable --now docker.service docker.socket`

## Install NVIDIA Driver and NVIDIA Container Toolkit

As of writing, the production NVIDIA driver is version 535, while bleeding edge is at 545.

### NVIDIA Driver

```bash
sudo apt install -y nvidia-driver-535
sudo reboot
```

### NVIDIA Container Toolkit

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey -o /tmp/nvidia-gpgkey

sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg /tmp/nvidia-gpgkey

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list -o /tmp/nvidia-list
```

As root (sudo won't work for this one)

```bash
su -
sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
    /tmp/nvidia-list > /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Then get out of your root shell.

```bash
sudo apt update -y
sudo apt install -y nvidia-container-toolkit
sudo reboot
```

### Configure NVIDIA Docker Runtime

`sudo nvidia-ctk runtime configure --runtime=docker`

Now to edit the nvidia container runtime config. Uncomment the line __#no-cgroups=false__ and set it to true, like this __no-cgroups=true__

I promise I will configure cgroups later so that we can run these containers rootless without having to modify `/etc/nvidia-container-runtime/config.toml`

`sudo vim /etc/nvidia-container-runtime/config.toml`

### Drop into the NVIDIA CUDA 12.2.0 Container

Now for a sanity check. If we can run the base image and access _nvidia-smi_, then our hard work paid off.

#### Passing ALL GPUs because We Don't Care

`docker run --gpus all -it nvidia/cuda:12.2.0-base-ubuntu22.04 bash`

You _should_ drop into a root shell. Run `nvidia-smi` and if you are greeted with information about your GPU, YOU DID IT!

#### Passing Only a Specific GPU into the Container

Notice that your gpu(s) are identified as /dev/nvidia{0-n}.

`ls /dev/nvidia*`

My machine has a single GPU in it right now, so my GPU is __/dev/nvidia0__, that is gpu device 0.

`docker run --gpus device=0 -it nvidia/cuda:12.2.0-base-ubuntu22.04 bash`