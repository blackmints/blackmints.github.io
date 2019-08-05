---
title: Building openbabel(pybel) for python3 within docker
tags:
  - openbabel
  - pybel
  - docker
---
I was supposed to use openbabel for converting molecular representations but I encountered difficulty in openbabel(pybel) installation.
Below, I describe how to make docker image with openbabel(pybel) working with python3 by building it inside. I used similar approach to the docerfile for rdkit. [link](https://blackmints.github.io/dockerfile-for-tensorflow-and-rdkit/)

### Building openbabel from source
Now the `pip install openbabel` for python3 crashes, we have to build it by providing PYTHON_EXECUTABLE on cmake.
```bash
# Build openbabel with python3
FROM debian:stretch AS rdkit-build-env

RUN apt-get update \
 && apt-get install -yq --no-install-recommends \
    cmake \
    wget \
    libcairo2-dev \
    libeigen3-dev \
    python3-dev \
    swig \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

ARG OPENBABEL_VERSION=openbabel-2-4-1
RUN wget --quiet https://github.com/openbabel/openbabel/archive/${OPENBABEL_VERSION}.tar.gz \
 && tar -xzf ${OPENBABEL_VERSION}.tar.gz \
 && mv openbabel-${OPENBABEL_VERSION} openbabel \
 && rm ${OPENBABEL_VERSION}.tar.gz

RUN mkdir /openbabel/build
WORKDIR /openbabel/build

RUN cmake \
  -D PYTHON_BINDINGS=ON \
  -D RUN_SWIG=ON \
  -D PYTHON_EXECUTABLE=/usr/bin/python3 \
  -D PYTHON_INCLUDE_DIR=/usr/include/python3.5 \
  -D CMAKE_INSTALL_PREFIX=/usr \
  -D CMAKE_BUILD_TYPE=Release \
  ..

RUN make -j $(nproc) \
 && make install
```
Important note is that if you don't install swig and build openbabel, it does not fail but returns successful. However, if you try to import openbabel or pybel the import succeeds, but modules inside them does not work.


### Copying openbabel from build-env to run-env
As I don't want packages used for building openbabel when actually working with openbabel, I copied openbabel from the environment for building to actual environment for running.
```bash
# Copy openbabel from openbabel-build-env
COPY --from=openbabel-build-env /usr/lib/libopenbabel* /usr/lib/
COPY --from=openbabel-build-env /usr/lib/openbabel /usr/lib/openbabel
COPY --from=openbabel-build-env /usr/lib/cmake/openbabel2/ /usr/lib/cmake/openbabel2/
COPY --from=openbabel-build-env /usr/share/openbabel /usr/share/openbabel
COPY --from=openbabel-build-env /usr/include/openbabel-2.0 /usr/include/openbabel-2.0
COPY --from=openbabel-build-env /usr/lib/python3/dist-packages/openbabel.py /usr/lib/python3/dist-packages/
COPY --from=openbabel-build-env /usr/lib/python3/dist-packages/_openbabel.so /usr/lib/python3/dist-packages/
COPY --from=openbabel-build-env /usr/lib/python3/dist-packages/pybel.py /usr/lib/python3/dist-packages/
COPY --from=openbabel-build-env /usr/lib/libinchi* /usr/lib/
COPY --from=openbabel-build-env /usr/include/inchi /usr/include/inchi
```

The entire dockerfile is as below.
```bash
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
    swig \
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

# Build openbabel in python3
WORKDIR /
ARG OPENBABEL_VERSION=openbabel-2-4-1
RUN wget --quiet https://github.com/openbabel/openbabel/archive/${OPENBABEL_VERSION}.tar.gz \
 && tar -xzf ${OPENBABEL_VERSION}.tar.gz \
 && mv openbabel-${OPENBABEL_VERSION} openbabel \
 && rm ${OPENBABEL_VERSION}.tar.gz

RUN mkdir /openbabel/build
WORKDIR /openbabel/build

RUN cmake \
  -D PYTHON_BINDINGS=ON \
  -D RUN_SWIG=ON \
  -D PYTHON_EXECUTABLE=/usr/bin/python3 \
  -D PYTHON_INCLUDE_DIR=/usr/include/python3.5 \
  -D CMAKE_INSTALL_PREFIX=/usr \
  -D CMAKE_BUILD_TYPE=Release \
  ..

RUN make -j $(nproc) \
 && make install

FROM tensorflow/tensorflow:latest-gpu-py3 AS rdkit-env

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

# Copy openbabel installation from rdkit-build-env
COPY --from=rdkit-build-env /usr/lib/libopenbabel* /usr/lib/
COPY --from=rdkit-build-env /usr/lib/openbabel /usr/lib/openbabel
COPY --from=rdkit-build-env /usr/lib/cmake/openbabel2/ /usr/lib/cmake/openbabel2/
COPY --from=rdkit-build-env /usr/share/openbabel /usr/share/openbabel
COPY --from=rdkit-build-env /usr/include/openbabel-2.0 /usr/include/openbabel-2.0
COPY --from=rdkit-build-env /usr/lib/python3/dist-packages/openbabel.py /usr/lib/python3/dist-packages/
COPY --from=rdkit-build-env /usr/lib/python3/dist-packages/_openbabel.so /usr/lib/python3/dist-packages/
COPY --from=rdkit-build-env /usr/lib/python3/dist-packages/pybel.py /usr/lib/python3/dist-packages/
COPY --from=rdkit-build-env /usr/lib/libinchi* /usr/lib/
COPY --from=rdkit-build-env /usr/include/inchi /usr/include/inchi
```
