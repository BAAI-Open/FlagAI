#  Pre-training distributed environment setup
- [Pre-training distributed environment setup](#pre-training-distributed-environment-setup)
- [一.  Docker](#一--docker)
  - [1.install docker](#1install-docker)
  - [2.Docker source change](#2docker-source-change)
  - [3.Install the graphics card driver (skip if installed)](#3install-the-graphics-card-driver-skip-if-installed)
  - [4.Configure NVIDIA-docker source：](#4configure-nvidia-docker-source)
  - [5.Make dockerfile](#5make-dockerfile)
    - [a. Pull the NVIDIA basic image and create a temporary folder (in the container, delete the image after it is created)](#a-pull-the-nvidia-basic-image-and-create-a-temporary-folder-in-the-container-delete-the-image-after-it-is-created)
    - [b.Configure apt installation source and install some common basic packages of Linux](#bconfigure-apt-installation-source-and-install-some-common-basic-packages-of-linux)
    - [c. Install the latest version of GIT (create an image clone installation package)](#c-install-the-latest-version-of-git-create-an-image-clone-installation-package)
    - [d. Install Mellanox OFED. Due to network problems, it is recommended to download the installation package locally and then execute dockerfile](#d-install-mellanox-ofed-due-to-network-problems-it-is-recommended-to-download-the-installation-package-locally-and-then-execute-dockerfile)
    - [e. Install nv_peer_mem](#e-install-nv_peer_mem)
    - [f. Install openmpi, You need to install the libevent dependency package first](#f-install-openmpi-you-need-to-install-the-libevent-dependency-package-first)
    - [g.Install python](#ginstall-python)
    - [h.Install magma-cuda](#hinstall-magma-cuda)
    - [i.Configuration path](#iconfiguration-path)
    - [j.Install some packages](#jinstall-some-packages)
    - [k.Install mpi4py (need to download to local installation, pip installation may report an error due to version compatibility)](#kinstall-mpi4py-need-to-download-to-local-installation-pip-installation-may-report-an-error-due-to-version-compatibility)
    - [l.Install pytorch, the version can be replaced, need to download locally first. Installation is easy to be terminated due to network speed. Some sub packages may be terminated during the download process of pytorch git clone. You can try few more times.](#linstall-pytorch-the-version-can-be-replaced-need-to-download-locally-first-installation-is-easy-to-be-terminated-due-to-network-speed-some-sub-packages-may-be-terminated-during-the-download-process-of-pytorch-git-clone-you-can-try-few-more-times)
    - [m.Install apex](#minstall-apex)
    - [n.Install deepspeed](#ninstall-deepspeed)
    - [o.Install NCCL(optional)](#oinstall-nccloptional)
    - [p.Configure network port, public key and SSH](#pconfigure-network-port-public-key-and-ssh)
  - [6.Build docker image](#6build-docker-image)
    - [a.Method 1. Pull image](#amethod-1-pull-image)
    - [b.Method 2. Build image](#bmethod-2-build-image)
- [二. Build containers at each machine node](#二-build-containers-at-each-machine-node)
- [三. Mutual trust mechanism setting](#三-mutual-trust-mechanism-setting)
  - [1. The default docker image for public key has been generated when it is created. If it does not exist, enter flow on the shell](#1-the-default-docker-image-for-public-key-has-been-generated-when-it-is-created-if-it-does-not-exist-enter-flow-on-the-shell)
  - [2.The public key file generated by each node container](#2the-public-key-file-generated-by-each-node-container)
  - [3.login without password](#3login-without-password)
  - [4.test](#4test)
- [四.  Distributed training test](#四--distributed-training-test)
  - [a.Configure hostfile (~/SSH/config and V100-1 in the hostfile correspondence):](#aconfigure-hostfile-sshconfig-and-v100-1-in-the-hostfile-correspondence)
  - [b. Configure glm files. Each node is configured with code and data. The path is required to be the same (you can also access cloud shared files together)](#b-configure-glm-files-each-node-is-configured-with-code-and-data-the-path-is-required-to-be-the-same-you-can-also-access-cloud-shared-files-together)
  - [c. cmd](#c-cmd)

# 一.  Docker

## 1.install docker

```shell
# Since the docker version in the apt official library in Ubuntu may be relatively low,
# uninstall the old version with the following command line
apt-get remove docker docker-engine docker-ce docker.io

# Update apt package index
apt-get update

# Execute the following command to enable apt to use the repository through HTTPS protocol
apt-get install -y apt-transport-https ca-certificates curl software-properties-common

# Add the GPG key officially provided by docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Setting up a stable repository
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Update apt package index again
apt-get update

# Install the latest version of docker-ce
apt-get install -y docker-ce
```

## 2.Docker source change

(https://xxxx.mirror.aliyuncs.com) is your own docker source

```shell
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxxx.mirror.aliyuncs.com"]
}
EOF

systemctl daemon-reload
systemctl restart docker
```

## 3.Install the graphics card driver (skip if installed)

```shell
# Check for NVIDIA drive
dpkg --list | grep nvidia-*
# or 'cat /proc/driver/nvidia/version'

# If the NVIDIA driver is not installed, it needs to be installed
# Execute Ubuntu drivers devices to view the recommended driver version
ubuntu-drivers devices
# If 'ubuntu-drivers' not found', run 'apt-get install ubuntu-drivers-common'

# Install the recommended driver version
apt-get install nvidia-driver-[recommended version]

# Verify successful installation
nvidia-smi
# If "NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver. Make sure that the latest NVIDIA driver is installed and running."
# Execute 'apt install dkms'
# Check version 'ls /usr/src | grep nvidia'
# 'dkms install -m nvidia -v + [version]'
# Note: you may need to restart the server after the installation is complete

```
## 4.Configure NVIDIA-docker source：

```shell
# Add source
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

# Install nvidia-docker2 and dependencies. During installation, select "default"
apt-get update
apt-get install -y nvidia-docker2
````

Modify /etc/docker/daemon.json，Add relevant information

```text
"runtimes": {
   "nvidia": {
       "path": "/usr/bin/nvidia-container-runtime",
       "runtimeArgs": []
   }
}
```
final content of /etc/docker/daemon.json

```json
{
 "registry-mirrors": ["https://xxxx.mirror.aliyuncs.com"],
 "runtimes": {
     "nvidia": {
         "path": "/usr/bin/nvidia-container-runtime",
         "runtimeArgs": []
     }
  }
}
```

reboot docker service

```shell
systemctl daemon-reload
systemctl restart docker
```

## 5.Make dockerfile

### a. Pull the NVIDIA basic image and create a temporary folder (in the container, delete the image after it is created)

```dockerfile
#pull base image
FROM nvidia/cuda:10.2-devel-ubuntu18.04 
#maintainer
MAINTAINER deepspeed <gqwang@baai.ac.cn>

##############################################################################
#Temporary Installation Directory
##############################################################################
ENV STAGE_DIR=/tmp
RUN mkdir -p ${STAGE_DIR}
```

### b.Configure apt installation source and install some common basic packages of Linux

```dockerfile
##############################################################################
#Installation/Basic Utilities
##############################################################################
RUN  sed -i s@/archive.ubuntu.com/@/mirrors.tuna.tsinghua.edu.cn/@g /etc/apt/sources.list
RUN  sed -i s@/security.ubuntu.com/@/mirrors.tuna.tsinghua.edu.cn/@g /etc/apt/sources.list
RUN apt-get update && \
    DEBIAN_FRONTEND="noninteractive" apt-get install -y --no-install-recommends \
        software-properties-common build-essential autotools-dev \
        nfs-common pdsh \
        cmake g++ gcc \
        curl wget vim tmux emacs less unzip \
        htop iftop iotop ca-certificates openssh-client openssh-server \
        rsync iputils-ping net-tools sudo \
        llvm-9-dev libsndfile-dev \
        libcupti-dev \
        libjpeg-dev \
        libpng-dev \
        screen jq psmisc dnsutils lsof musl-dev systemd
```      
      
### c. Install the latest version of GIT (create an image clone installation package)

```dockerfile
##############################################################################
#Installation Latest Git
##############################################################################
RUN add-apt-repository ppa:git-core/ppa -y && \
    apt-get update && \
    apt-get install -y git && \
    git --version
```

### d. Install Mellanox OFED. Due to network problems, it is recommended to download the installation package locally and then execute dockerfile

```dockerfile
##############################################################################
#install Mellanox OFED
#dwonload from  https://www.mellanox.com/downloads/ofed/MLNX_OFED-5.1-2.5.8.0/MLNX_OFED_LINUX-5.1-2.5.8.0-ubuntu18.04-x86_64.tgz
##############################################################################
RUN apt-get install -y libnuma-dev  libnuma-dev libcap2
ENV MLNX_OFED_VERSION=5.1-2.5.8.0
COPY MLNX_OFED_LINUX-${MLNX_OFED_VERSION}-ubuntu18.04-x86_64.tgz ${STAGE_DIR}
RUN cd ${STAGE_DIR} && \
    tar xvfz MLNX_OFED_LINUX-${MLNX_OFED_VERSION}-ubuntu18.04-x86_64.tgz && \
    cd MLNX_OFED_LINUX-${MLNX_OFED_VERSION}-ubuntu18.04-x86_64 && \
    PATH=/usr/bin:$PATH ./mlnxofedinstall --user-space-only --without-fw-update --umad-dev-rw --all -q && \
    cd ${STAGE_DIR} && \
    rm -rf ${STAGE_DIR}/MLNX_OFED_LINUX-${MLNX_OFED_VERSION}-ubuntu18.04-x86_64*
```  

### e. Install nv_peer_mem

```dockerfile
##############################################################################
#Install nv_peer_mem
##############################################################################
#COPY nv_peer_memory ${STAGE_DIR}/nv_peer_memory (without net)
############try for more times #################

ENV NV_PEER_MEM_VERSION=1.1
ENV NV_PEER_MEM_TAG=1.1-0
RUN git clone https://github.com/Mellanox/nv_peer_memory.git --branch ${NV_PEER_MEM_TAG} ${STAGE_DIR}/nv_peer_memory
RUN cd ${STAGE_DIR}/nv_peer_memory && \
    ./build_module.sh && \
    cd ${STAGE_DIR} && \
    tar xzf ${STAGE_DIR}/nvidia-peer-memory_${NV_PEER_MEM_VERSION}.orig.tar.gz && \
    cd ${STAGE_DIR}/nvidia-peer-memory-${NV_PEER_MEM_VERSION} && \
    apt-get update && \
    apt-get install -y dkms && \
    dpkg-buildpackage -us -uc && \
    dpkg -i ${STAGE_DIR}/nvidia-peer-memory_${NV_PEER_MEM_TAG}_all.deb 
```

### f. Install openmpi, You need to install the libevent dependency package first

```dockerfile
###########################################################################
#Install libevent && OPENMPI
#https://www.open-mpi.org/software/ompi/v4.0/
##############################################################################
ENV OPENMPI_BASEVERSION=4.0
ENV OPENMPI_VERSION=${OPENMPI_BASEVERSION}.5
COPY openmpi-4.0.5.tar.gz  ${STAGE_DIR}
COPY libevent-2.0.22-stable.tar.gz  ${STAGE_DIR}
RUN cd ${STAGE_DIR} && \
    tar zxvf libevent-2.0.22-stable.tar.gz && \
    cd libevent-2.0.22-stable && \
    ./configure --prefix=/usr && \
    make && make install
RUN cd ${STAGE_DIR} && \
    tar --no-same-owner -xzf openmpi-4.0.5.tar.gz && \
    cd openmpi-${OPENMPI_VERSION} && \
    ./configure --prefix=/usr/local/openmpi-${OPENMPI_VERSION} && \
    make -j"$(nproc)" install  && \
    ln -s /usr/local/openmpi-${OPENMPI_VERSION} /usr/local/mpi && \
    #Sanity check:
    test -f /usr/local/mpi/bin/mpic++ && \
    cd ${STAGE_DIR} && \
    rm -r ${STAGE_DIR}/openmpi-${OPENMPI_VERSION}
ENV PATH=/usr/local/mpi/bin:${PATH} \
    LD_LIBRARY_PATH=/usr/local/lib:/usr/local/mpi/lib:/usr/local/mpi/lib64:${LD_LIBRARY_PATH}
#Create a wrapper for OpenMPI to allow running as root by default
RUN mv /usr/local/mpi/bin/mpirun /usr/local/mpi/bin/mpirun.real && \
    echo '#!/bin/bash' > /usr/local/mpi/bin/mpirun && \
    echo 'mpirun.real --allow-run-as-root --prefix /usr/local/mpi "$@"' >> /usr/local/mpi/bin/mpirun && \
    chmod a+x /usr/local/mpi/bin/mpirun
```
  
### g.Install python

```dockerfile
###########################################################################
#Install python
##############################################################################
ARG PYTHON_VERSION=3.8
RUN curl -o ~/miniconda.sh https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-latest-Linux-x86_64.sh && \
     chmod +x ~/miniconda.sh && \
     ~/miniconda.sh -b -p /opt/conda && \
     rm ~/miniconda.sh && \
     /opt/conda/bin/conda install -y python=$PYTHON_VERSION numpy pyyaml scipy ipython mkl mkl-include ninja cython typing
```
  
### h.Install magma-cuda

```dockerfile
###########################################################################
#Install magma-cuda
##############################################################################
COPY magma-cuda102-2.5.2-1.tar.bz2   ${STAGE_DIR} 
RUN  cd ${STAGE_DIR} && \
     /opt/conda/bin/conda install -y -c pytorch --use-local magma-cuda102-2.5.2-1.tar.bz2  && \
     /opt/conda/bin/conda clean -ya
####optional#####
#RUN  /opt/conda/bin/conda install -y -c pytorch  magma-cuda102  && \
#/opt/conda/bin/conda clean -ya
```
  
### i.Configuration path

```dockerfile
###########################################################################
#Export path
##############################################################################
ENV PATH /opt/conda/bin:$PATH
RUN echo "export PATH=/opt/conda/bin:\$PATH" >> /root/.bashrc
RUN pip install --upgrade pip setuptools
RUN wget https://tuna.moe/oh-my-tuna/oh-my-tuna.py && python oh-my-tuna.py
```
  
### j.Install some packages

```dockerfile
###########################################################################
#Install some Packages
##############################################################################
RUN pip install psutil \
                yappi \
                cffi \
                ipdb \
                h5py \
                pandas \
                matplotlib \
                py3nvml \
                pyarrow \
                graphviz \
                astor \
                boto3 \
                tqdm \
                sentencepiece \
                msgpack \
                requests \
                pandas \
                sphinx \
                sphinx_rtd_theme \
                sklearn \
                scikit-learn \
                nvidia-ml-py3 \
                nltk \
                rouge \
                filelock \
                fasttext \
                rouge_score \
                cupy-cuda102\
                setuptools==60.0.3
```
  
### k.Install mpi4py (need to download to local installation, pip installation may report an error due to version compatibility)

```dockerfile
##############################################################################
#Install mpi4py
##############################################################################
RUN apt-get update && \
 apt-get install -y mpich
COPY mpi4py-3.1.3.tar.gz ${STAGE_DIR}
RUN cd ${STAGE_DIR} && tar zxvf mpi4py-3.1.3.tar.gz && \
 cd mpi4py-3.1.3 &&\
 python setup.py build && python setup.py install
```

### l.Install pytorch, the version can be replaced, need to download locally first. Installation is easy to be terminated due to network speed. Some sub packages may be terminated during the download process of pytorch git clone. You can try few more times.

```dockerfile
##############################################################################
#PyTorch
#clone (may be time out because of the network problem)
#RUN git clone --recursive https://github.com/pytorch/pytorch --branch v1.8.1 /opt/pytorch
#RUN cd /opt/pytorch && git checkout -f v1.8.1 && \
#git submodule sync && git submodule update -f --init --recursive
##############################################################################
ENV TORCH_CUDA_ARCH_LIST="6.0 6.1 7.0+PTX"

COPY opt/pytorch /opt/pytorch
ENV NCCL_LIBRARY=/usr/lib/x86_64-linux-gnu
ENV NCCL_INCLUDE_DIR=/usr/include
RUN cd /opt/pytorch && TORCH_NVCC_FLAGS="-Xfatbin -compress-all" \
    CMAKE_PREFIX_PATH="$(dirname $(which conda))/../" USE_SYSTEM_NCCL=1 \
    pip install -v . && rm -rf /opt/pytorch



##############################################################################
#Install vision
#RUN git clone https://github.com/pytorch/vision.git /opt/vision
##############################################################################
COPY vision /opt/vision
RUN cd /opt/vision  && pip install -v . && rm -rf /opt/vision
ENV TENSORBOARDX_VERSION=1.8
RUN pip install tensorboardX==${TENSORBOARDX_VERSION}
```

### m.Install apex

```dockerfile
###########################################################################
#Install apex
###########################################################################
#RUN git clone https://github.com/NVIDIA/apex ${STAGE_DIR}/apex
COPY apex ${STAGE_DIR}/apex
RUN cd ${STAGE_DIR}/apex && pip install -v --no-cache-dir --global-option="--cpp_ext" --global-option="--cuda_ext" ./ \
    && rm -rf ${STAGE_DIR}/apex
```

### n.Install deepspeed

```dockerfile
 ############################################################################
#Install deepSpeed
#############################################################################
RUN pip install  py-cpuinfo
RUN apt-get install -y libaio-dev
ENV TORCH_CUDA_ARCH_LIST="6.0 6.1 7.0+PTX"
RUN git clone https://github.com/microsoft/DeepSpeed.git ${STAGE_DIR}/DeepSpeed
#COPY DeepSpeed ${STAGE_DIR}/DeepSpeed
RUN cd ${STAGE_DIR}/DeepSpeed &&  \
    git checkout . && \
    DS_BUILD_OPS=1 ./install.sh -r
RUN rm -rf ${STAGE_DIR}/DeepSpeed
RUN python -c "import deepspeed; print(deepspeed.__version__)"
```

### o.Install NCCL(optional)

```dockerfile
############################################################################
#Install nccl
#############################################################################

#COPY  nccl-local-repo-ubuntu1804-2.9.6-cuda10.2_1.0-1_amd64.deb ${STAGE_DIR}
#RUN cd ${STAGE_DIR} &&\
#sudo dpkg -i nccl-local-repo-ubuntu1804-2.9.6-cuda10.2_1.0-1_amd64.deb &&\
#sudo apt install -y   libnccl2 libnccl-dev
#RUN apt install -y --allow-downgrades --no-install-recommends --allow-change-held-packages  libnccl2=2.9.6-1+cuda10.2 libnccl-dev=2.9.6-1+cuda10.2
#ENV NCCL_VERSION=2.9.6
```

### p.Configure network port, public key and SSH

```dockerfile
#############################################################################
#Set SSH Config
#############################################################################
RUN apt-get install openssh-server

ARG SSH_PORT=6001
#RUN echo 'root:NdjeS+-4gEPmq}D' | chpasswd
#Client Liveness & Uncomment Port 22 for SSH Daemon
RUN echo "ClientAliveInterval 30" >> /etc/ssh/sshd_config
RUN mkdir -p /var/run/sshd && cp /etc/ssh/sshd_config ${STAGE_DIR}/sshd_config && \
    sed "0,/^#Port 22/s//Port 22/" ${STAGE_DIR}/sshd_config > /etc/ssh/sshd_config
RUN cat /etc/ssh/sshd_config > ${STAGE_DIR}/sshd_config && \
    sed "0,/^Port 22/s//Port ${SSH_PORT}/" ${STAGE_DIR}/sshd_config > /etc/ssh/sshd_config && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/g' /etc/ssh/sshd_config
EXPOSE ${SSH_PORT}
#Set SSH KEY
RUN printf "#StrictHostKeyChecking no\n#UserKnownHostsFile /dev/null" >> /etc/ssh/ssh_config && \
 ssh-keygen -t rsa -f ~/.ssh/id_rsa -N "" && cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys && \
   chmod og-wx ~/.ssh/authorized_keys
CMD service ssh start
```


## 6.Build docker image

### a.Method 1. Pull image

```shell
# remote pull
docker pull deepspeed/cuda102
# local pull
docker load --input deepspeed-cuda102.tar.gz
```
### b.Method 2. Build image

```shell
docker build -f cuda102.dockerfile  -t deepspeed/cuda102:1221 .
# cuda102.dockerfile Refer to the production process of dockerfile file
```   
# 二. Build containers at each machine node

```shell
# Create a container (NVIDIA-docker),
# hostname=the host name inside the container
# network = host share with the several machine
# ipc = host  This is required for cluster training.
# shm_size shared memory, name container external name,
# --gpus specifies GPU, multi data volume:
# -v local folder: folder in container -v local folder: folder in container -v local folder: folder in container deepspeed / cuda102:1221 image name: tag
nvidia-docker run -id  --hostname=glm_dist16  --network=host --ipc=host --shm-size=16gb --name=glm_dist16   --gpus '"device=0,1,2,3"' -v /data1/docker/containers:/data  deepspeed/cuda102:1221
```

```shell
# pull image
docker pull nvidia/cuda:cuda-verison-runtime-ubuntu-version
# pull image example
docker pull nvidia/cuda:10.1-runtime-ubuntu18.04
# create container (simple docker), port number：-p 22:22 -p 80:80 -p 8080:8080，multi data folder ：-v folder:folder -v folder:folder -v folder:folder
docker run -id --name=container_name -p Host_port: container_port -e TZ=Asia/Shanghai -v host_folder: container_folder image:version
# create container (simple docker) example
docker run -id --name=test -p 80:80 -e TZ=Asia/Shanghai -v /data:/data mysql:5.7
# create container（nvidia-docker），Support multi mapping，multi port number：-p 22:22 -p 80:80 -p 8080:8080，multi data folder：-v folder:folder -v folder:folder -v folder:folder
nvidia-docker run -id --name=container_name -p Host_port: container_port -e TZ=Asia/Shanghai --shm-size=size -v host_folder: container_folder image:version
# create container（nvidia-docker）example
docker run -id --name=test -p 80:80 -e TZ=Asia/Shanghai --shm-size=8gb -v /data:/data nvidia/cuda:10.1-runtime-ubuntu18.04
# Enter container（simple docker）
docker exec -it container_name /bin/bash
# Enter container（nvidia-docker）
nvidia-docker exec -it container_name /bin/bash
# View existing images
docker images
# delete image
docker rmi image_name/image_id
# View running containers
docker ps
# View history container, including running and closed
docker ps -a
# Stop a running container
docker stop container_name/container_id
# Delete the container. If the container to be deleted is running, you need to stop it first and then delete it
docker rm container_name/container_id
```

# 三. Mutual trust mechanism setting

## 1. The default docker image for public key has been generated when it is created. If it does not exist, enter flow on the shell

```shell
 ssh-keygen -t rsa -C "example@com.cn"
```

## 2.The public key file generated by each node container

~/.ssh/id_rsa.pub
The contents in are collected and synchronized to the files of each machine
~/.ssh/authorized_keys

## 3.login without password

Configure each node container port : vi /etc/ssh/sshd_config , (Cancel the port annotation and set the value to unify the nodes. Different containers need different ports)
Configure each node as follows host 文件 ：vi ~/.ssh/config, (Copy the host login information of each node and synchronize to each node)

```text
Host V100-1
    Hostname 172.31.32.29
    Port 6001
    User root
Host V100-2
    Hostname 172.31.32.40
    Port 6002
    User root
```

## 4.test

```shell
ssh V100-1
```

# 四.  Distributed training test

## a.Configure hostfile (~/SSH/config and V100-1 in the hostfile correspondence):

```text
V100-1 slots=4
V100-2 slots=4
V100-3 slots=4
.....
```

## b. Configure glm files. Each node is configured with code and data. The path is required to be the same (you can also access cloud shared files together)

## c. cmd

```shell
bash config/start_scripts/generate_block.sh  config/config_tasks/model_blocklm_large_chinese.sh
```

