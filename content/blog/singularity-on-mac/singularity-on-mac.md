---
title: Singularity on Mac, Reproducible Research and Lesson Learned
description: How to set up Singularity on a Mac using Lima, build portable containers, and overcome common pitfalls like architecture mismatches
date: 2025-03-03
tags: [software, tips]
---

Remember the time when you want to install multiple software to reproduce someone else's research? What does it remind you of? Pain? Exhaustion from hours of Googling and troubleshooting? Only to face the famous "version compatibility" issues even after you're sure everything right according to the tutorial. Ah, the joys of science and its ever-looming "reproducibility crisis"!

Luckily we have containerisation technology these days! As written on the [IBM website](https://www.ibm.com/topics/containerization) "Containerisation is the packaging of software code with just the operating system (OS) libraries and dependencies required to run the code to create a single lightweight executable—called a container—that runs consistently on any infrastructure.". 

Started as a way to make the deployment process easier, containerisation has emerged as a valuable tool for ensuring reproducible research, so instead of letting the user install the software themselves and make them find the scripts (stored don't know where), giving them this so called 'container' streamline this process.

Docker and Singularity are among the most popular containerisation platform (Well technically Docker is more popular, haven't heard of Singularity till my PI mentioned it). However since I'm largely working with HPC environments, Singularity is preferred as it's designed for such purpose and more importantly it's open-source (Long live open-source!).

Singularity is built for Linux, so you have another problem if you're using Windows or Mac, in which you have to use virtual machine (VM) to run it. In this post, I'm going to share my experience dealing with Singularity on my Mac, from installing to building a singularity image file (`.sif`, end-product, the thing that you share to other people for reproducible research) and the lessons that I learned. So let's dive in.

As I said, since Singularity is primarily created for Linux, Mac users need to use virtual machines. Several options exist to create virtual machines on Mac, including [Vagrant](https://www.vagrantup.com/), [Lima](https://lima-vm.io/) [OrbStack](https://orbstack.dev/]) (which unfortunately is not very open-source). In my case I'm using Lima as I found it to be straightforward and aligned with the [Singularity installation guide](https://docs.sylabs.io/guides/4.1/admin-guide/installation.html#mac). 

Download [Homebrew](https://brew.sh/) if you haven't had it installed
```bash
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
$ (echo; echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"') >> $HOME/.profile
$ eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
```

Install Lima
```bash
$ brew install lima
```

Running lima is pretty straightforward and it comes with [various distros](https://lima-vm.io/docs/templates/) that you can use. In this case we're going to use the default [singularity-ce.yml](https://raw.githubusercontent.com/sylabs/singularity/main/examples/lima/singularity-ce.yml) template that is provided in the docs.

```yml
# SingularityCE on Alma Linux 9
# 
# Usage:
#
#   $ limactl start ./singularity-ce.yml
#   $ limactl shell singularity-ce singularity run library://alpine

images:
- location: "https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2"
  arch: "x86_64"
- location: "https://repo.almalinux.org/almalinux/9/cloud/aarch64/images/AlmaLinux-9-GenericCloud-latest.aarch64.qcow2"
  arch: "aarch64"

mounts:
- location: "~"
- location: "/tmp/lima"

containerd:
  system: false
  user: false

provision:
- mode: system
  script: |
    #!/bin/bash
    set -eux -o pipefail
    dnf -y install --enablerepo=epel singularity-ce squashfs-tools-ng

probes:
- script: |
    #!/bin/bash
    set -eux -o pipefail
    if ! timeout 30s bash -c "until command -v singularity >/dev/null 2>&1; do sleep 3; done"; then
      echo >&2 "singularity is not installed yet"
      exit 1
    fi
  hint: See "/var/log/cloud-init-output.log" in the guest

message: |
  To run `singularity` inside your lima VM:
    $ limactl shell {{Name}} singularity run library://alpine
```

TLDR; It builds an AlmaLinux and Singularity installed with it

You can update the template above to include more options, for example since the default will only spare 4gb of memory adding `memory: "8GiB"` will spare 8 instead of 4, adding `writable` options below the mount points will let you write files in those locations (do it at your own risk, also mind that if `writable` not being set to `true` the files in the mounted directory become `read-only`)
```yaml
...

memory: "8GiB"

mounts:
- location: "~"
writable: true
- location: "/tmp/lima"
writable: true

...
```

running the command below will start your lima vm (Select `Proceed with the current configuration option` when prompted)

```bash
$ limactl start ./singularity-ce.yml
```

When you run the command below, you will see that your `singularity-ce` instance is already running

```bash
$ limactl list
```

To stop and remove/delete the instance you can use the following command respectively
```
$ limactl stop singularity-ce
$ limactl remove singularity-ce
$ limactl delete singularity-ce
```

Now we got our `singularity-ce` running, next step is to create the singularity container, let's say I want to create container that contains softwares from different programming language. The command below will let you enter the VM interactively 

```bash
$ limactl shell singularity-ce
[user@lima-singularity-ce]$ 
```

You can also directly run singularity, for example the command below will run the Alpine Linux image from the Sylabs cloud library
```bash
$ limactl shell singularity-ce singularity run library://alpine
```

Since we will be creating the Singularity container image ourselves, we won't be downloading it from the Sylabs cloud library but instead we will write something called `.def` file. `.def` file or definition file is a recipe to build container image with Singularity, it's just a Singularity headers and bunch of shell commands that will be executed to create the image container. This guides from [Singularity docs](https://docs.sylabs.io/guides/latest/user-guide/definition_files.html) is very helpful in explaining what we can put in the `.def` file.

Below is a minimal script to create a container with `pandas` and `seaborn` installed

`test.def`
```def
Bootstrap: docker
From: ubuntu:22.04

%files
	/Users/hariesramdhani/Documents/requirements.txt /requirements.txt

%post
	export DEBIAN_FRONTEND=noninteractive
	
	apt-get clean all && \
	apt-get update && \
	apt-get upgrade -y && \
	apt-get install -y \
	autoconf \
	autogen \
	build-essential \
	curl \
	libbz2-dev \
	libcurl4-openssl-dev \
	libssl-dev \
	libxml2-dev \
	zlib1g-dev \
	python3-dev \
	python3-pip

	pip install -r /requirements.txt
```

`/Users/hariesramdhani/Documents/requirements.txt`
```txt
pandas
seaborn
```

To create image from the above `.def` file, all you have to do is run the following command (make sure you already activated the interactive mode, notice the `[user@lima-singularity-ce]` before the $)
```
[user@lima-singularity-ce]$ singularity build --fakeroot test.sif test.def
```

The command below build the `test.sif` from the `test.def` and you either need to run it as `sudo` or using the `--fakeroot` argument. Usually it fails when I'm using `sudo`, this is why I'm using `--fakeroot`. The `.sif` file will appear once you successfully build it. 

Well it seems easy and straightforward, doesn't it? It does, but in practice, it can be more complicated when you try to build a more complex system, like for example my `.def` file will contain different softwares from different programming language different way to install and "different" everything on an ARM64 M2 Macbook. Here I am sharing the lessons I learned when building singularity image for my project.

**My ARM != Your AMD**
Mac transitioned to Apple silicon which means the CPU architecture of your Macbook M series is ARM64. Why is this information important? The thing with Singularity is that it's not architecture agnostic, so when you build your singularity image file on your Mac (ARM64) and try to run it on an AMD64 HPC you will get the following error;
```
$ singularity run test.sif
FATAL:   could not open image /path/to/dir/test.sif: the image's architecture (arm64) could not run on the host's (amd64)
```

Can't we just the create the Alpine Linux on our Lima VM to be AMD64 in the first place? Yes, technically by adding the `arch` parameter and set it to `x86_64` we will make our Lima VM architecture to be AMD64. 
```yaml
...

arch: "x86_64"

images:
- location: "https://repo.almalinux.org/almalinux/9/cloud/x86_64/images/AlmaLinux-9-GenericCloud-latest.x86_64.qcow2"
  arch: "x86_64"

...
```

This sounds straightforward and expected to be working but it didn't for me, but luckily after some hacking here and there I found a getaway! What I did for my case was to build an Ubuntu vm then install Singularity inside it.

Can simply create a new instance using the default `ubuntu-lts` and when prompted with option don't forget to update the yaml file
```bash
$ limactl start ubuntu-lts
```

If the default editor is vim, press `i` to insert, edit it to add the following and `esc` followed `w+q` and `enter` to save
```yml
...

# Change the architecture
arch: "x86_64"

# Upgrade the memory
memory: "8GiB"
mounts:

# Make the mounted point writable
- location: "~"
  writable: true
- location: "/tmp/lima"
  writable: true

...
```

It may take more time than the usual to create the instance (most likely because of the different architecture). Once you're done, enter the interactive mode to install singularity.

```bash
$ singularity shell ubuntu-lts
```

And run the following code to install Singularity and its dependencies, change the GO version (`1.13.15`) and Singularity version (`v3.6.3`) according to your liking, as seen in the [installation guide](https://github.com/apptainer/singularity/issues/5099#issuecomment-814563244))

```bash
sudo apt-get update && \
sudo apt-get install -y build-essential \
libseccomp-dev pkg-config squashfs-tools cryptsetup

sudo rm -r /usr/local/go

export VERSION=1.13.15 OS=linux ARCH=amd64  # change this as you need

wget -O /tmp/go${VERSION}.${OS}-${ARCH}.tar.gz https://dl.google.com/go/go${VERSION}.${OS}-${ARCH}.tar.gz && \
sudo tar -C /usr/local -xzf /tmp/go${VERSION}.${OS}-${ARCH}.tar.gz

echo 'export GOPATH=${HOME}/go' >> ~/.bashrc && \
echo 'export PATH=/usr/local/go/bin:${PATH}:${GOPATH}/bin' >> ~/.bashrc && \
source ~/.bashrc

curl -sfL https://install.goreleaser.com/github.com/golangci/golangci-lint.sh |
sh -s -- -b $(go env GOPATH)/bin v1.21.0

mkdir -p ${GOPATH}/src/github.com/sylabs && \
cd ${GOPATH}/src/github.com/sylabs && \
git clone https://github.com/sylabs/singularity.git && \
cd singularity

git checkout v3.6.3

cd ${GOPATH}/src/github.com/sylabs/singularity && \
./mconfig && \
cd ./builddir && \
make && \
sudo make install

singularity version
```

I tried to `run` the singularity image file that I `built` using the above VM on an AMD64 HPC and it works successfully, the `build`ing process however takes longer time than usual!

**NIH HPC Singularity def files are your friend**
Whether you want to create a cool, complex def file or need to know how to install some stuff (yes R packages I'm looking at you!), you can always refer to [NIH HPC Singularity def files collection](https://github.com/NIH-HPC/singularity-def-files/tree/master) on GitHub (as there aren't many on Google search), they have it all! Just use the Github advanced search `repo:NIH-HPC/singularity-def-files {keyword}` to find specific type of commands, also technically you can find more by searching all over GitHub with the `path:*.def "Bootstrap:"` search keyword, but the NIH HPC was suffice for my work.

**The power  of `script`**
Building singularity (`singularity build`) image file with a complex `.def` file may take a while (by a while I meant a whole hour) and when you run the command thousands of lines of installation progress appear (Yes R packages I'm looking at you again), troubleshooting can be very hard when you have to scroll through all of those thousands of lines and looking for where the error occurred. Fortunately we have `script` command on linux, `script {filename_to_save_the_logs}` will record all of the things that are written on the terminal whether it's an output, an error message or literally anything, it then will save it for you when you run the `exit` command. You can then open it using your favourite text editor and `CTRL+F` or `command+F`.

**Captain we need more memory**
Setting up the right `memory` is very vital as you don't want your vm to hang when building your Singularity image files. Happened several times for me before I decided to increase the memory size.

**Escaping R and its package dependency hell**
I had a very hard time installing R and some of its packages, what worked for me was to follow the recipes from [NIH HPC Singularity def files collection](https://github.com/NIH-HPC/singularity-def-files/tree/master) that contain the keyword  `Rscript`, `install_github`.  While the installation worked it didn't solve the dependencies problem especially when installing the package using `install_github` and that package requires Bioconductor packages, earlier today I had a supervisory meeting with Mike (my PI) and he came up with this nice idea to actually scrape the `DESCRIPTION` file for each package from the Github repo and install them first before installing the package. I'll show how I do it on a separate blog post!

So that concludes my writing, hopefully this can be helpful for all of the researchers that are going to the same thing.