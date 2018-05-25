Docker Swarm Tutorial
=====================
*by s.m.Lee*

## Swarm Tutorial ##

그럼 이제 Docker CLI를 통해 swarm을 만들고, swarm에 service application을 deploy 해보자.

*공식 문서를 참고 하였고, 조금 다른 방향으로 진행해 보겠다*

진행 순서는 아래와 같다.

1. intializing a cluster of Docker Engines in swarm mode
2. adding nodes to the swarm
3. deploying application services to the swarm
4. managing the swarm once you have everything running

이 튜토리얼을 진행하기 위해서는 

1. 5개의 linux host

   - 하나의 manager와, 4개의 worker node로 구성 할 것이다.

2. Docker Engine 1.12 or later installed 

   - install Docker Engine on Linux machines
     리눅스 기반의 OS를 사용중이면, 공식 document의 install instruction을 보면 됨.
     이미 docker 가 설치되어져 있다면 skip하자

3. the IP address of the manager machine

   - IP address는 고정 ip로 할당되어져 있어야 한다.
   - swarm에 속한 모든 node는 IP에 의해 manager에 connect 되어야한다.

4. open ports between the hosts

   - TCP port 2377 - for cluster management communications
   - TCP and UDP port 7946 - for communication among nodes
   - UDP port 4789 for overlay network traffic

이 port들은 docker 설치시 자동으로 열리게 되는걸로 알고있다. 

혹시 모르니 $netstat -tnlp | grep LISTEN 으로 확인해보자


이제 Swarm을 만들어보자.
Docker Engine daemon이 host machine에서 작동중인지 확인해라

공식 문서에는 docker-machine을 이용해서 manager node를 실행시키고 ssh로 machine에 접속하라는 형태로 쓰여 있는데

linux host라면 docker-machine 구성이 필요 없다.
