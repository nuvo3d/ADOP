ARG CUDA_VERSION=11.1
ARG UBUNTU_VERSION=20.04
FROM nvidia/cudagl:${CUDA_VERSION}-devel-ubuntu${UBUNTU_VERSION}

ARG CPU_SIZE=4
ARG PYTHON_VERSION=3.9

USER root

ENV DEPENDENCIES="/dependencies"
WORKDIR ${DEPENDENCIES}

ENV DEBIAN_FRONTEND noninteractive


#----------------------------------------------
# Dependencies
#----------------------------------------------

RUN apt-get -y update                && \
    apt-get -y upgrade               && \
    apt-get install -y                  \
    software-properties-common      \
    sudo                            \
    cmake                           \
    build-essential                 \
    wget                            \
    curl                            \
    git                             \
    swig                            \
    bzip2                           \
    xorg-dev                        \
    x11-apps                        \
    mesa-utils                      \
    ffmpeg

# CMAKE
RUN apt purge -y --auto-remove cmake && \
    wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null | gpg --dearmor - | sudo tee /etc/apt/trusted.gpg.d/kitware.gpg >/dev/null && \
    apt-add-repository 'deb https://apt.kitware.com/ubuntu/ bionic main' && \
    apt-get update && \
    apt-get install -y cmake
        
# C++ 20
RUN add-apt-repository ppa:ubuntu-toolchain-r/test && \
    apt update && \
    apt-get install -y gcc-9 g++-9 && \
    update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 60 --slave /usr/bin/g++ g++ /usr/bin/g++-9 && \
    update-alternatives --config gcc

# TBB
RUN cd ${DEPENDENCIES} && \
    git clone https://github.com/wjakob/tbb.git && \
    cd tbb/build && \
    cmake .. && \
    make -j${CPU_SIZE} && \
    make install 

ENV CUDA_ROOT /usr/local/cuda-${CUDA_VERSION}
ENV PATH $PATH:$CUDA_ROOT/bin
ENV LD_LIBRARY_PATH $LD_LIBRARY_PATH:$CUDA_ROOT/lib64:$CUDA_ROOT/lib:/usr/local/nvidia/lib64:/usr/local/nvidia/lib
ENV LIBRARY_PATH /usr/local/nvidia/lib64:/usr/local/nvidia/lib:/usr/local/cuda/lib64/stubs:/usr/local/cuda/lib64:/usr/local/cuda/lib$LIBRARY_PATH

# Anaconda
ENV PATH=/opt/conda/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
RUN /bin/sh -c set -x &&     apt-get update --fix-missing &&     apt-get install -y --no-install-recommends         bzip2         ca-certificates         git         libglib2.0-0         libsm6         libxcomposite1         libxcursor1         libxdamage1         libxext6         libxfixes3         libxi6         libxinerama1         libxrandr2         libxrender1         mercurial         openssh-client         procps         subversion         wget     && apt-get clean     && rm -rf /var/lib/apt/lists/* &&     UNAME_M="$(uname -m)" &&     if [ "${UNAME_M}" = "x86_64" ]; then         ANACONDA_URL="https://repo.anaconda.com/archive/Anaconda3-2021.11-Linux-x86_64.sh";         SHA256SUM="fedf9e340039557f7b5e8a8a86affa9d299f5e9820144bd7b92ae9f7ee08ac60";     elif [ "${UNAME_M}" = "s390x" ]; then         ANACONDA_URL="https://repo.anaconda.com/archive/Anaconda3-2021.11-Linux-s390x.sh";         SHA256SUM="1504e9259816c5804eff1304fe7e339517b9fc1a08bfd991bc525a7efb6568f1";     elif [ "${UNAME_M}" = "aarch64" ]; then         ANACONDA_URL="https://repo.anaconda.com/archive/Anaconda3-2021.11-Linux-aarch64.sh";         SHA256SUM="4daacb88fbd3a6c14e28cd3b37004ed4c2643e2b187302e927eb81a074e837bc";     elif [ "${UNAME_M}" = "ppc64le" ]; then         ANACONDA_URL="https://repo.anaconda.com/archive/Anaconda3-2021.11-Linux-ppc64le.sh";         SHA256SUM="7eb6a95925ee756240818599f8dcbba7a155adfb05ef6cd5336aa3c083de65f3";     fi &&     wget "${ANACONDA_URL}" -O anaconda.sh -q &&     echo "${SHA256SUM} anaconda.sh" > shasum &&     sha256sum --check --status shasum &&     /bin/bash anaconda.sh -b -p /opt/conda &&     rm anaconda.sh shasum &&     ln -s /opt/conda/etc/profile.d/conda.sh /etc/profile.d/conda.sh &&     echo ". /opt/conda/etc/profile.d/conda.sh" >> ~/.bashrc &&     echo "conda activate base" >> ~/.bashrc &&     find /opt/conda/ -follow -type f -name '*.a' -delete &&     find /opt/conda/ -follow -type f -name '*.js.map' -delete &&     /opt/conda/bin/conda clean -afy # buildkit

ENV CC=gcc-9
ENV CXX=g++-9
ENV CUDAHOSTCXX=g++-9


#----------------------------------------------
# ADOP
#----------------------------------------------

ENV WORKSPACES="/workspaces"
ENV PROJECT="ADOP"

RUN mkdir ${WORKSPACES} && \
    cd ${WORKSPACES} && \
    git clone https://github.com/nuvo3d/${PROJECT}.git && \
    cd ${PROJECT} && \
    git submodule update --init --recursive --jobs 0

WORKDIR ${WORKSPACES}/${PROJECT}

RUN ./create_environment.sh

RUN echo "source activate adop" > ~/.bashrc
ENV PATH ${CONDA}/bin:$PATH

RUN ./install_pytorch.sh

ENV CONDA=/opt/conda/envs/adop

RUN mkdir build && \
    cd build && \
    cmake -DCMAKE_PREFIX_PATH="${CONDA}/lib/python${PYTHON_VERSION}/site-packages/torch/;${CONDA}" \
    -DCMAKE_CUDA_COMPILER=${CONDA}/bin/nvcc .. && \
    make -j${CPU_SIZE}


#-------------------------------------------------------------
#       Post Processing
#-------------------------------------------------------------

ENV USER=dock
ENV GROUP=sudo

RUN useradd -ms /bin/bash ${USER} && \
    usermod -aG ${GROUP} ${USER}

RUN cd ${WORKSPACES} && \
    chown -R dock:dock ./ADOP

USER root

RUN apt-get autoremove -y && \
    apt-get autoclean -y && \
    rm -rf /var/lib/apt/lists/*

# Resolve authorization problem
RUN echo "${USER} ALL=(ALL) NOPASSWD: ALL" \
    >/etc/sudoers.d/${USER} && \
    chmod 0440 /etc/sudoers.d/${USER}

USER dock