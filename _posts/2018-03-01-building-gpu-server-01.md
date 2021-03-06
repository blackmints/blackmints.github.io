---
title: GPU 서버 세팅기: 우분투 설치 및 세팅
tags:
  - cuda
  - tensorflow
  - ubuntu
---

연구실에 tensorflow용 gpu 워크스테이션을 새로 마련해서 세팅을 하게되었다. 생각보다 까다로운 부분이 많았어서 겸사겸사 기록으로 남겨둔다.

## 1. 부팅디스크 제작
제일 흔하고 무난한 ubuntu 16.04 LTS로 설치했다. 사용하는 데스크탑이 우분투여서 부팅디스크를 어떻게 쓰는지 몰라 찾아봤는데 "시동 디스크 만들기" 라는 내장 프로그램을 쓰면 된다. 윈도우라면 몇몇 흔히 쓰이는 프로그램이 있으니 그걸 사용하면 ok. 8GB usb에 포맷 후 만들었다.

## 2. 우분투 설치
시작부터 의외의 난관에 봉착했다. Usb 부팅을 하면 우분투를 설치하겠냐는 화면으로 넘어가는데, 설치를 누르면 검은 화면이 되고는 아무 반응이 없다. 구글링을 해보니 nouveau가 범인이었다. 이걸 끄려면 F6을 눌러서 뜨는 리스트에 nomodeset을 스페이스해서 체크해주면 된다. 또한 나는 겪지 않았는데 secure boot나 fast boot의 함정도 있다고 하니 조심하는게 좋을듯.

참고: [http://blog.neonkid.xyz/66](http://blog.neonkid.xyz/66), [http://ejklike.github.io/2017/03/05/install-ubuntu-16.04-with-nvidia-gpu.html](http://ejklike.github.io/2017/03/05/install-ubuntu-16.04-with-nvidia-gpu.html)

## 3. 고정 IP 세팅
학교 인터넷망은 전부 고정 ip를 할당받아 사용하게 되어있는데다 어짜피 서버이니 고정 ip를 받을 수 밖에 없다. 우분투 데스크탑 GUI로는 고정 ip 세팅을 해본적이 있는데 CLI로 해보긴 처음이었다. 미리 신청해놓은 ip를 가지고 세팅하는데 생각보다 까다로웠다.

~~~bash
# 랜카드 인터페이스 이름 확인
$ ifconfig -a

# 랜포트가 여러개라 랜선이 꽂힌 인터페이스를 확인하고 싶을땐
$ /sbin/ethtool 장치명 | grep Link

# 만약 dhcp를 사용할 경우
$ dhclient 장치명

# 만약 고정ip를 사용할 경우
$ ifconfig 장치명 아이피 넷마스크 서브넷마스크 up
$ vi /etc/network/interfaces

# auto 장치명
# iface 장치명 inet static
# address *.*.*.*
# network *.*.*.0
# gateway *.*.*.1
# netmask 255.255.255.0
# broadcast *.*.*.255

# 수정한 세팅을 적용
$ service networking restart
~~~

세팅을 다 했는데도 외부로 ping이 안나가는 문제가 있어서 고민했는데, 학교 망에서 NIC 인증이 되지 않은 ip를 기본적으로 막아놓고 있어서 그런거였다. 홈페이지서 인증을 신청하니 정상적으로 됨. Ping을 테스트할때는 self -> gatewat -> DNS -> 외부 host 순으로 테스트.

## 4. 방화벽, SSH
~~~bash
# SSH 설치 및 세팅
$ apt-get update
$ apt-get install ssh
$ vi /etc/ssh/sshd_config

# 방화벽 세팅
$ ufw allow 22
$ ufw allow 22/tcp
$ ufw allow ssh
~~~

여기까지 하면 기본적인 ssh 및 서버 세팅은 완료된다.
