---
title: Ubuntu 18.04 LTS GPU 서버 세팅
tags:
  - ubuntu
  - tensorflow
  - cuda
  - poetry
---

새 gpu를 구매한 김에 ubuntu를 18.04로 올리고 CUDA 10을 깔고자 서버를 다시 세팅했다.

## 1. 부팅디스크 제작
Ubuntu 18.04 LTS 이미지를 다운받아 rufus로 usb 부팅디스크를 만들었다.

## 2. 우분투 설치
혹시나 설치과정에서 검은 화면이 뜨고 먹통이 된다면 nouveau 문제일 확률이 높다. 이 경우, F6을 눌러 리스트에 있는 nomodeset을 체크해주고 설치하면 된다.

## 3. 고정 IP 세팅
서버는 보통 DHCP대신 고정IP로 설정하여 원격으로 사용하니 세팅을 해주자.
```bash
# 랜카드 설정 확인
$ ifconfig

# 랜선이 꽂힌 인터페이스를 확인하고 싶을때
$ /sbin/ethtool "장치명" | grep Link

# 고정 IP 설정
$ ifconfig 장치명 아이피 넷마스크 서브넷마스크 up
$ vi /etc/network/interfaces
# auto "장치명"
# iface "장치명" inet static
# address XXX.XXX.XXX.XXX
# network XXX.XXX.XXX.0
# gateway XXX.XXX.XXX.1
# netmask 255.255.255.0
# broadcast XXX.XXX.XXX.255

# DNS 설정 확인, 구글 DNS 설정
$ nslookup server
$ vi /etc/resolvconf/resolv.conf.d/tail
# nameserver 8.8.8.8

# 수정한 세팅을 적용
$ service networking restart

# DNS 서버 동작 확인
$ nslookup www.google.com 8.8.8.8
```

## 4. 방화벽, SSH 설치
### SSH 설치 및 세팅
```bash
$ apt-get update
$ apt-get install ssh
$ vi /etc/ssh/sshd_config
```

### 방화벽 세팅
```bash
$ ufw allow 22
$ ufw allow 22/tcp
$ ufw allow ssh
```

## 5. NVIDIA 드라이버 설치
드라이버 버전에 따라 지원되는 GPU, CUDA 버전이 다르니 주의가 필요하다.
- Turing  >= 410.48
- Volta >= 384.111
- CUDA 10.1 (10.1.105)	>= 418.39
- CUDA 10.0 (10.0.130)	>= 410.48
- CUDA 9.2 (9.2.88) >= 396.26
- CUDA 9.1 (9.1.85)	>= 390.46
- CUDA 9.0 (9.0.76)	>= 384.81

한편, 드라이버 설치가 18.04부터 쉬워졌다고 한다.
```bash
$ sudo add-apt-repository ppa:graphics-drivers/ppa
$ sudo apt-get update
$ ubuntu-drivers devices | grep "nvidia-driver"
$ sudo apt-get install --no-install-recommends nvidia-driver-430
$ sudo reboot
$ nvidia-smi
```

## 5.1 Docker 사용시 9로 진행

## 6. CUDA 10.0 설치
드라이버 버전과 호환되는 CUDA를 설치하자. Tensorflow의 요구 버전은 아래와 같다.

| Tensorflow | Python | cuDNN | CUDA |
|------------|--------|-------|------|
|1.13.1         | 2.7, 3.3-3.7 | 7.4  | 10.0  |
|1.12.0 ~ 1.5.0 | 2.7, 3.3-3.6 | 7    | 9     |

```bash
$ sudo apt-get install --no-install-recommends \
    cuda-10-0 \
    libcudnn7=7.4.1.5-1+cuda10.0  \
    libcudnn7-dev=7.4.1.5-1+cuda10.0
```

## 7. Path 설정
```bash
$ export PATH=$PATH:/usr/local/cuda-10.0/bin
$ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda-10.0/lib64   
$ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/extras/CUPTI/lib64
```

## 8. Poetry 설치
Poetry를 설치하기 전에 ubuntu의 python 버전을 3.6으로 설정하자.
```bash
$ python --version
$ which python
$ ls /usr/bin/ | grep python
$ sudo update-alternatives --config python
$ sudo update-alternatives --install /usr/bin/python python /usr/bin/python3.6 1
$ python --version
```

이제 Poetry를 설치하자.
```bash
$ curl -sSL https://raw.githubusercontent.com/sdispater/poetry/master/get-poetry.py | python
# source $HOME/.poetry/env
$ poetry —version
# Bash completion
$ poetry completions bash > /etc/bash_completion.d/poetry.bash-completion
```

## 9. Docker 설치
Docker를 사용하면 드라이버만 깔고 CUDA 등을 설치할 필요가 없다.
```bash
$ curl -fsSL https://get.docker.com/ | sudo sh
$ sudo usermod -aG docker 사용자
# re-login
$ docker version
```
GPU 가속을 사용한 tensorflow를 활용하려면 nvidia-docker를 사용해서 실행해야만 한다. 이 과정은 nvidia 드라이버가 설치되어있어야만 하니 꼭 설치한 뒤에 하자.
```bash
$ curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
$ distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
$ curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list
$ sudo apt-get update
$ sudo apt-get install -y nvidia-docker2
$ sudo pkill -SIGHUP dockerd
$ docker run --runtime=nvidia --rm nvidia/cuda:9.0-base nvidia-smi
```
이제 tensorflow-gpu의 최신 이미지를 받아와서 실행해보자
```bash
$ docker login
$ docker pull tensorflow/tensorflow:latest-gpu-py3
$ docker images
$ docker run --runtime=nvidia -it -p 6006:6006 --name tf1.13.1 \
  tensorflow:latest-gpu-py3
```
