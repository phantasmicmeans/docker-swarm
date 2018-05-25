Docker Swarm
==============

by S.M.Lee

## Orchestration  ##

 서버 오케스트레이션을 간단하게 정의하면 "여러 서버와 서비스를 편리하게 관리하는 작업" 이라 볼 수 
있을 것이다. 뭐 간단하게 말하면 이런 형태지만 scheduling, clustering, service discovery, loggin, monitoring 등
여러 작업을 해야한다.

많은 수의 서버들이 전부 Docker Container로 띄워져 있고, 이 Docker Host들을 
엮어서 마치 하나의 서버처럼 다루고 많은 것을 자동화 한다면 관리적인 측면에 있어 좋지 않을까?

여기서는 Clustering Tool, 즉 Container Orchestration Tool(대표적 Kubernetes) 하나인 Docker Swarm에 대해 알아보겠다.

본래 Orchestration Tool은 수백, 수천대의 서버들을 관리하기 위한 목적으로 만들어졌기에 
적은 양의 서버를 관리하기 위해 Orchestration Tool을 도입하는 것은 비용적인 부분이나, 인적 자원 낭비라고도 볼 수 있지만..

Docker Swarm은 다른 Tool에 비해 구축 비용도 적고, 사용하거나 관리하기 쉽기에 Swarm을 사용해 보려고 한다.

## Docker Swarm이란? ## 

![image](https://user-images.githubusercontent.com/20153890/40529551-694425bc-6030-11e8-8eb3-0164551e020b.png)

Docker Host Pool을 단일 가상 Docker Host로 전환해 준다. 
공식 문서는 **"The cluster management and orchestration features embedded in the
Docker Engine are built using swarmkit"** 라고 말한다.
 
한마디로 Docker Engine에 **Embedded** 되어있는 cluster management, 오케스트레이션 툴이다.

공식 문서에는 Docker verion이 1.12.0 이상이면 standalone swarm을 사용할 수 
있다고 나와있다. 따라서 update를 recommend 한다고 말하고 있으므로 낮은 버전의 docker를 사용중이라면
시작하기 전 docker version을 update하자.

먼저 각각의 docker host들은 swarm manager 또는 worker(service) 역할을 수행할 수 있다.
그리고 Swarm은 swarm mode에서 실행되는 manager와 worker(swarm service)로 동작하는 
여러 Docker hosts를 포함한다.

swarm service를 만들때 optimal state(복제본, 스토리지 리소스, 네트워크, 노출 포트) 등을
정의할 수 있고, Docker 는 이 state를 유지하려고 한다.

## Docker Swarm의 Component ##
Docker Swarm은 여러 요소들로 구성되는데 이 요소들에 대해 간단하게 짚어보고 넘어가자.

*구성 요소들에 대한 설명은 공식문서를 참고하여 작성하였다.*

 ### 1. swarm ### 
 
 위에서도 설명했지만, 분산된 컨테이너를 실행 할 수 있는 클러스터이다.

 ### 2. node ###

node란 swarm에 참여하는 docker engine의 인스턴스이다. 또한 이를 docker node라고 볼 수 있다. 

manager node와 worker node로 구성되고, manager node는 swarm cluster를 관리하는 node이다.
worker node는 manager node의 명령을 받아 container를 서비스하는 node라 이해하면 된다.
하나의 computer나 cloud 서버에서 하나 이상의 node를 실행 가능 하지만,
일반적으로 swarm 배포에는 여러 클라우드 서버에 분산된 docker node가 포함된다

하나의 application을 swarm에 deploy하려면 service 정의를 manager에게 전달해야 한다.
그리고 manager node는 task라는 작업 단위를 worker node에게 전달한다.
 manager node는 원하는 swarm 상태를 유지하는데 필요한 orchestration 및
cluster management 기능을 수행한다.
또한 manager node는 orchestration 을 수행할 node들의 leader를 뽑는다.

위에서 설명한대로, manager node는 worker node에게 task를 전달하고 실행시킨다.
default로, manager node는 service를 worker node에 실행하도록 하지만 manager-only하게 service를 실행하도록 구성할 수 있다.

 ### 3. Service and Task ###

service는, manager node 또는 worker node에서 실행되는 task의 정의이다. 
service를 create할때 container에서 사용할 image와 실행할 command를 지정한다.
replicate된 service 에서는 swarm manager가 설정한 비율에 따라 node사이에 특정 수의
replicate task를 deploy한다.

그리고 global service는, swarm cluster의 모든 node에서 service에 대한 하나의 task를 실행한다
모든 node가 특정 service의 task를 할당 받는다는 얘기이다. 

Manager node는 service scale에 설정된 replica 수 에 따라 task를 worker node에 할당한다.
task가 node에 할당되면 다른 node로 이동할 수 없다.

 ### 4. Load balancing ###

swarm manager는 Load balancing을 통해 외부에서 사용할수 있도록 service를 swarm에 노출한다.

swarm manager는 자동으로 service의 PublishedPort를 지정하거나, 따로 PublishedPort를 지정할 수 있다.
port를 따로 지정하지 않으면 30000 - 32767 범위의 port를 할당한다.

swarm mode는 내부 DNS 구성요소를 가지고 있어, DNS 항목에 있는 각 서비스를 자동으로 할당한다.
swarm manager는 내부 load balancing을 통해 service의 DNS 이름에 따라 
cluster내의 service간 요청을 분산한다.

지금까지 Docker Swarm의 개념과, 구성요소에 대해 살펴보았다.

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

