Docker Swarm Tutorial
=====================
*by s.m.Lee*

Orchestration과 Docker Swarm에 대해 알아보았고, Component 또한 어느정도 이해가 되었을 것이라 생각한다.

그럼 이제 Swarm을 구성하여 보고, service를 deploy 하자.

# Swarm Tutorial #

그럼 이제 Docker CLI를 통해 swarm을 만들고, swarm에 service application을 deploy 해보자.

*공식 문서를 참고 하였고, 조금 다른 방향으로 진행해 보겠다*

**NOTE**

진행 순서는 아래와 같다.
##
 1. intializing a cluster of Docker Engines in swarm mode
 2. adding nodes to the swarm
 3. deploying application services to the swarm
 4. managing the swarm once you have everything running
##

이 튜토리얼을 진행하기 위해서는 

### 1. 5개의 linux host

   - 하나의 manager node와, 4개의 worker node로 구성 할 것이다.

### 2. Docker Engine 1.12 or later installed 

   - install Docker Engine on Linux machines
   - 리눅스 기반의 OS를 사용중이면, 공식 document의 install instruction을 보면 된다.
     이미 docker 가 설치되어져 있다면 skip하자

### 3. the IP address of the manager machine

   - IP address는 고정 ip로 할당되어져 있어야 한다.
   - swarm에 속한 모든 node는 IP에 의해 manager에 connect 되어야한다.

### 4. open ports between the hosts

   - TCP port 2377 - for cluster management communications
   - TCP and UDP port 7946 - for communication among nodes
   - UDP port 4789 for overlay network traffic

이 2377, 7946 port는 설치시 자동으로 열려 있을 것이다(내 경우엔..). 혹시 모르니 $netstat -tnlp | grep LISTEN 으로 확인해보자

![image](https://user-images.githubusercontent.com/20153890/40533128-b3342792-603c-11e8-8988-5a7d6ab31280.png)


이제 Swarm을 만들어보자.

공식 문서에는 docker-machine을 이용해서 manager node를 실행시키고 ssh로 machine에 접속하라는 형태로 쓰여 있는데

linux host라면 docker-machine 구성이 필요 없다.

1. swarm을 구성해 보자 

> - $docker swarm init --advertise-addr <MANAGER-IP>
<MANAGER-IP> 부분에 SWARM HOST로 사용할 IP를 입력하면 된다.

![image](https://user-images.githubusercontent.com/20153890/40534399-e1249cbe-6040-11e8-8e85-414edb37eeb1.png)

--advertise-addr flag는 manager node address가 192.168.10.174를 가지도록 한다.

그리고 새로운 node가 swarm에 join할수 있는 command를 포함하고 있다. node는 manager나 worker로 swarm에 join할 것이고, 
중간에 나와있는 docker swarm join --token ~~ command를 node로 지정할 server에서 실행시키면 된다.

