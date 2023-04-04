# Description

The docker image is preinstalled with mp3d library, and a specific version of huggingface transformer. If you want to train and run VLN model in mp3d environment, you can safely follow the below steps; however, if you want a newer version of huggingface transformer, or if you want to add other libraries like weights and bias (wandb), you may need to recreate the docker image. (Since singularity does not allow you to modify the image permanently after creation, you can install libraries temporarily, but you need to reinstall them every time you start the container, :))

Contact me if you have a better way <yangziji@oregonstate.edu>

# Prepare for Singularity
```
# docker hub login
export APPTAINER_DOCKER_USERNAME=""
export APPTAINER_DOCKER_PASSWORD=""
VLN_HAMT_DIR="/nfs/hpc/sw/zijiao/unitVLN/VLN-HAMT/"
DATA_DIR="/nfs/hpc/share/yangziji/repos/"

# you need to get the matterport data from https://niessner.github.io/Matterport/
MATTERPORT_DATA_DIR="/nfs/hpc/share/yangziji/data/matterport/images/v1/scans"

# it binds data dir from outside environment into your container environment
export APPTAINER_BIND="$MATTERPORT_DATA_DIR:/usr/src/env_drop/data/v1/scans, $DATA_DIR/Recurrent-VLN-BERT:/usr/src/Recurrent-VLN-BERT,$DATA_DIR/crafty_env/output:/usr/src/crafty, /nfs/hpc/share/yangziji/data/RR:/usr/src/RR, $DATA_DIR/neoteric:/usr/src/neoteric, $DATA_DIR/speaker-fo:/usr/src/speaker-fo, $VLN_HAMT_DIR:/usr/src/VLN-HAMT, /nfs/hpc/share/yangziji/data:/usr/src/data, /nfs/hpc/sw/zijiao/unitVLN/CLIP-ViL/CLIP-ViL-VLN:/usr/src/clip-vln, /nfs/hpc/sw/zijiao/cache:/usr/src/cache"

```
# Get image
```singularity pull docker://tianshup/tianshu2zijiao:20210526```

# Usage
```
# the --env-file is to modify/add pythonpath to container environemnt  
singularity shell --env-file ./vlnunit_env --nv ./tianshu2zijiao_20210526.sif
```
in `vlnunit_env` file, I have the following content, (...build/ store the mp3d library)
```
PYTHONPATH=/usr/src/env_drop/build/:/usr/src/VLN-HAMT
```
# Docker file
```
# Matterport3DSimulator requires nvidia gpu with driver 396.37 or higher
FROM nvidia/cudagl:9.2-devel-ubuntu18.04
WORKDIR /usr/src/env_drop
# install cudnn
ENV CUDNN_VERSION 7.6.4.38
ENV PYTHONIOENCODING=utf-8
LABEL com.nvidia.cudnn.version="${CUDNN_VERSION}"

RUN apt-get update && apt-get install -y --no-install-recommends \
    libcudnn7=$CUDNN_VERSION-1+cuda9.2 \
libcudnn7-dev=$CUDNN_VERSION-1+cuda9.2 \
&& \
    apt-mark hold libcudnn7 && \
    rm -rf /var/lib/apt/lists/*


# install a few libraries to support both EGL and OSMESA options
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y git wget doxygen curl libjsoncpp-dev libepoxy-dev libglm-dev libosmesa6 libosmesa6-dev libglew-dev libopencv-dev python-opencv python3-setuptools python3-dev python3-pip git-all
RUN pip3 install opencv-python==4.1.0.25 torch==1.6 torchvision numpy pandas networkx Flask
RUN pip3 install boto3 requests tqdm lmdb ifcfg h5py regex pudb tensorboardX ipdb optuna wandb

# install latest cmake
ADD https://cmake.org/files/v3.12/cmake-3.12.2-Linux-x86_64.sh /cmake-3.12.2-Linux-x86_64.sh
RUN mkdir /opt/cmake
RUN sh /cmake-3.12.2-Linux-x86_64.sh --prefix=/opt/cmake --skip-license
RUN ln -s /opt/cmake/bin/cmake /usr/local/bin/cmake
RUN cmake --version

# added compile of simulator
COPY . /usr/src/env_drop/
RUN mkdir build
WORKDIR build
RUN cmake -DEGL_RENDERING=ON ..
RUN make -j8
# install correct version of transfomer
# RUN pip3 install git+git://github.com/huggingface/transformers.git@067923d3267325f525f4e46f357360c191ba562e
ENV PYTHONPATH=/usr/src/env_drop/build:/usr/src/Recurrent-VLN-BERT:/usr/src/env_drop/Oscar
WORKDIR /usr/src/env_drop/Oscar
RUN python3 setup.py build develop
WORKDIR /usr/src
# RUN pip3 install transformers==1.0.0
# set pythonpath
```