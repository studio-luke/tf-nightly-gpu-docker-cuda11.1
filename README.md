# Docker Setting for Multi-GPU

Workstation Environment:

- OS: Ubuntu 18.04
- NVIDIA driver version : 455.38
- CUDA version : 11.1
- Graphics Card: NVIDIA RTX 3090 x 4

## Building Docker Image

You can find the final version of dockerfile in the link below:

[`https://github.com/studio-luke/tf-nightly-gpu-docker-cuda11.1/blob/master/dockerfiles/gpu-jupyter-cuda11.1.Dockerfile`](https://github.com/studio-luke/tf-nightly-gpu-docker-cuda11.1/blob/master/dockerfiles/gpu-jupyter-cuda11.1.Dockerfile)

You might want to download the whole repository [`https://github.com/studio-luke/tf-nightly-gpu-docker-cuda11.1`](https://github.com/studio-luke/tf-nightly-gpu-docker-cuda11.1), because you also need files named 'bashrc', 'readme-for-jupyter.md' while building the image. This will be explained later.

- This dockerfile is based on tensorflow official GitHub: [`https://github.com/tensorflow/tensorflow/blob/master/tensorflow/tools/dockerfiles/dockerfiles/gpu-jupyter.Dockerfile`](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/tools/dockerfiles/dockerfiles/gpu-jupyter.Dockerfile) (2020.12.08)
- (This file is subject to change/update, so I saved current version of this dockerfile in [`https://github.com/studio-luke/tf-nightly-gpu-docker-cuda11.1/blob/master/dockerfiles/gpu-jupyter.Dockerfile`](https://github.com/studio-luke/tf-nightly-gpu-docker-cuda11.1/blob/master/dockerfiles/gpu-jupyter.Dockerfile))

The goal is to set docker image to have the environment below:

- CUDA version: 11.1
- Python version: 3.8.x
- Tensorflow version: nightly (2.5.x)

[1] Change CUDA version from 11.0 to 11.1

```docker
ARG CUDA=11.1
```

[2] Change libnvinfer version from 7.1.3 to newest 7.2.1

```docker
ARG LIBNVINFER=7.2.1-1
```

[3] Install Python3.8, instead of installing general Python 3 (which will install Python 3.6.x)

```docker
RUN apt-get update && apt-get install -y python3.8
RUN apt-get update && apt-get install -y python3-pip
```

[4] Replace all `python3` command to `python3.8`.

```docker
RUN python3.8 -m pip --no-cache-dir install --upgrade \
    pip \
    setuptools
...

RUN ln -s $(which python3.8) /usr/local/bin/python

...

RUN python3.8 -m pip install --no-cache-dir jupyter matplotlib

...

RUN python3.8 -m ipykernel.kernelspec
```

[5] (Optional) Install needed packages for our deep learning program

```docker
# Install needed packages for Our Deep learning program
RUN python3.8 -m pip --no-cache-dir install \
    tensorflow_datasets \
    opencv-python \
    pydicom
    
# Install tensorflow_examples
RUN apt-get install -y --no-install-recommends git
RUN python3.8 -m pip install -q git+https://github.com/tensorflow/examples.git

# Install library for cv2
RUN apt install -y libgl1-mesa-glx

# Install vim
RUN apt-get install -y vim
```

[6] (Optional) Install Jupyter Notebook dark theme

```docker
# Jupyter Themes
RUN python3.8 -m pip --no-cache-dir install jupyterthemes
RUN jt -t oceans16 -fs 115 -nfs 125 -tfs 115 -dfs 115 -ofs 115 -cursc r -cellw 80% -lineh 115 -altmd  -kl -T -N
```

[7] In my case, `tensorflow-gpu` required `/usr/local/cuda/lib64/libcusolver.so.10` file, but we had `/usr/local/cuda/lib64/libcusolver.so.11` . Generating link between them

```docker
# Make link to libcusolver.so.10, since it's required when registering GPU devices
RUN ln /usr/local/cuda/lib64/libcusolver.so.11 /usr/local/cuda/lib64/libcusolver.so.10
```

[8] Install tensorflow nightly version

```docker
# Options:
#   tensorflow
#   tensorflow-gpu
#   tf-nightly
#   tf-nightly-gpu
# Set --build-arg TF_PACKAGE_VERSION=1.11.0rc0 to install a specific version.
# Installs the latest version by default.

ARG TF_PACKAGE=tf-nightly-gpu
ARG TF_PACKAGE_VERSION=2.5.0.dev20201207
RUN python3.8 -m pip install --no-cache-dir ${TF_PACKAGE}${TF_PACKAGE_VERSION:+==${TF_PACKAGE_VERSION}}
```

[9] I found that installing some packages together lead to crash: `nbformat==4.4.0`, the log says they are finding compatible package, and the message is endlessly repeated. **Just to skip installing this prevented from error.** 

```docker
RUN python3.8 -m pip install --no-cache-dir jupyter_http_over_ws ipykernel==5.1.1
# Ignore installing nbformat==4.4.0. making Conflict Issue
```

The dockerfile is ALL SET! When you build the image, run `docker build` command in the same directory with `bashrc`, `readme-for-jupyter.md`, since dockerfile is trying to copy these files while building.

```bash
docker build -f ./dockerfiles/gpu-jupyter-cuda11.1.Dockerfile -t image-name:tag-name .
```

## Run the Docker Container

When you want to run this image, there're several options you should specify. Try running command below:

```bash
sudo docker run --rm \
		--runtime=nvidia \
		-v ~/notebooks:/tf/notebooks \
		--gpus all \
		--shm-size=32gb \
		-p 9999:8888 image-name:tag-name
```

- `--rm`: automatically remove the container file after it is stopped
- `--runtime=nvidia` / `--gpus all`: allowing docker image to use GPU hardwares
- `-v ~/notebooks:/tf/notebooks`: setting volumes for your docker container. With this option, any changes you made in `/tf/notebooks` in your docker container will be simultaneously synchronized with `~/notebooks` in your local machine
- `--shm-size`: setting shared memory size for docker container. This is important for docker container to use enough memory space. If this option is not specified, default shared memory size is 64MB.
- `-p`: setting port to access your jupyter notebook. with `9999:8888`, your jupyter notebook will run in port 8888 in your workstation, and you can access to the notebook through the link `localhost:9999`, if tunneling is well enabled.
- `image-name:tag-name`: the name:tag of the image you might want to run.
