---
title: Dockerfile for Tensorflow-gpu and RDKit
tags:
  - docker
  - tensorflow
  - rdkit
---
### Docker image for deep learning chemistry
As RDKit does not provide pip repository for installation on virtualenv, I planned to use docker image for environment setting for my project. By building rdkit from the source inside docker, rdkit packages are obtained and transferred to working docker image. Only solution other than docker is to use conda environment, as I know.

Dockerhub: https://hub.docker.com/r/harry0305/tensorflow-rdkit

### Core intentions
- Latest tensorflow-gpu (=1.13.1)
- Latest rdkit (=2019.03.2)

### Key Changes
- Upgrading rdkit version from 2018.09 to 2019.03 additionally requires libboost-iostreams.
- Changing the python verion PYTHON_INCLUDE_DIR=/usr/include/python3.5 to other breaks the build.
- Executing rdkit requires boost 1.62.0, however tensorflow/tensorflow:latest-gpu-py3 is built over ubuntu xenial. When apt-get install libboost, it installs 1.58.0 rather than 1.62.0. Therefore, I used personal PPA to obtain 1.62.0 for xenial, which is ppa:bkryza/onedata-deps-gcc7.
- When executed, libstdc++.so.6: version 'GLIBCXX_3.4.22' not found occurs. It is fixed by adding additional ppa:ubuntu-toolchain-r/test and installation of gcc-4.9, libstdc++6.

### References
- https://github.com/nyuge/rdkit-build
- https://github.com/mcs07/docker-rdkit
- https://launchpad.net/~bkryza/+archive/ubuntu/onedata-deps-gcc7
- https://github.com/lhelontra/tensorflow-on-arm/issues/13

---
```docker
FROM debian:stretch AS rdkit-build-env

RUN apt-get update \
 && apt-get install -yq --no-install-recommends \
    ca-certificates \
    build-essential \
    cmake \
    wget \
    libboost-dev \
    libboost-system-dev \
    libboost-thread-dev \
    libboost-serialization-dev \
    libboost-python-dev \
    libboost-regex-dev \
    libboost-iostreams-dev \
    libcairo2-dev \
    libeigen3-dev \
    python3-dev \
    python3-numpy \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

ARG RDKIT_VERSION=Release_2019_03_2
RUN wget --quiet https://github.com/rdkit/rdkit/archive/${RDKIT_VERSION}.tar.gz \
 && tar -xzf ${RDKIT_VERSION}.tar.gz \
 && mv rdkit-${RDKIT_VERSION} rdkit \
 && rm ${RDKIT_VERSION}.tar.gz

RUN mkdir /rdkit/build
WORKDIR /rdkit/build

# RDK_OPTIMIZE_NATIVE=ON assumes container will be run on the same architecture on which it is built
RUN cmake -Wno-dev \
  -D RDK_INSTALL_INTREE=OFF \
  -D RDK_INSTALL_STATIC_LIBS=OFF \
  -D RDK_BUILD_INCHI_SUPPORT=ON \
  -D RDK_BUILD_AVALON_SUPPORT=ON \
  -D RDK_BUILD_PYTHON_WRAPPERS=ON \
  -D RDK_BUILD_CAIRO_SUPPORT=ON \
  -D RDK_USE_FLEXBISON=OFF \
  -D RDK_BUILD_THREADSAFE_SSS=ON \
  -D RDK_OPTIMIZE_NATIVE=ON \
  -D PYTHON_EXECUTABLE=/usr/bin/python3 \
  -D PYTHON_INCLUDE_DIR=/usr/include/python3.5 \
  -D PYTHON_NUMPY_INCLUDE_PATH=/usr/lib/python3/dist-packages/numpy/core/include \
  -D CMAKE_INSTALL_PREFIX=/usr \
  -D CMAKE_BUILD_TYPE=Release \
  ..

RUN make -j $(nproc) \
 && make install

FROM tensorflow:latest-gpu-py3 AS rdkit-env

# Install runtime dependencies
RUN add-apt-repository ppa:ubuntu-toolchain-r/test
RUN add-apt-repository ppa:bkryza/onedata-deps-gcc7
RUN apt-get update \
 && apt-get install -yq --no-install-recommends \
    libboost-atomic1.62.0 \
    libboost-chrono1.62.0 \
    libboost-date-time1.62.0 \
    libboost-python1.62.0 \
    libboost-regex1.62.0 \
    libboost-serialization1.62.0 \
    libboost-system1.62.0 \
    libboost-thread1.62.0 \
    libboost-iostreams1.62.0 \
    libcairo2-dev \
    python3-dev \
    python3-numpy \
    python3-cairo \
    gcc-4.9 \
 && apt-get upgrade -yq libstdc++6 \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

# Copy rdkit installation from rdkit-build-env
COPY --from=rdkit-build-env /usr/lib/libRDKit* /usr/lib/
COPY --from=rdkit-build-env /usr/lib/cmake/rdkit/* /usr/lib/cmake/rdkit/
COPY --from=rdkit-build-env /usr/share/RDKit /usr/share/RDKit
COPY --from=rdkit-build-env /usr/include/rdkit /usr/include/rdkit
COPY --from=rdkit-build-env /usr/lib/python3/dist-packages/rdkit /usr/lib/python3/dist-packages/rdkit
```
