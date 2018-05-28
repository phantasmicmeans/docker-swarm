Docker Swarm Tutorial
=====================
*by s.m.Lee*

![image](https://user-images.githubusercontent.com/20153890/40535846-41728da2-6045-11e8-87c7-e301beac3c46.png)

Orchestration과 Docker Swarm에 대해 알아보았고, Component 또한 어느정도 이해가 되었을 것이라 생각한다.
그럼 이제 Swarm을 구성하여 보고, service를 deploy 해보자.

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


## Swarm

공식 문서에는 docker-machine을 이용해서 manager node를 실행시키고 ssh로 machine에 접속하라는 형태로 쓰여 있는데
linux host라면 docker-machine 구성이 필요 없다.

### 1. intializing a cluster of Docker Engines in swarm mode

> - $docker swarm init --advertise-addr [MANAGER-IP]

manager node로 사용할 linux host server에 위 command를 실행한다. [MANAGER-IP] 부분에 manager node 로 사용할 IP를 입력하면 된다.

![image](https://user-images.githubusercontent.com/20153890/40534751-0bfa467c-6042-11e8-80cb-4ad6b612a9ec.png)

--advertise-addr flag는 manager node가 자신의 address를 192.168.10.174로 게시하도록 한다.

실행 결과, 중간에 새로운 node가 swarm에 join할수 있는 command(token을 포함한)를 포함하고 있다. 또 다른 node는 manager나 worker로 swarm에 join할 수 있다. 여기서는 4개의 node들이 worker node로 swarm에 join할 것이므로 중간에 나와있는 **docker swarm join --token ~~ command**를 worker node로 지정할 server에서 실행시키면 된다.

### 2. adding nodes to the swarm

여기서는 4개의 worker node를 사용할 것 이므로 worker로 사용 예정중인 linux host server 4개에 각각 docker swarm join --token ~~ command를 실행하면 된다.
swarm에 join시킬 command를 잊었다면, $docker swarm join-token manager 를 manager node server에서 실행하면 확인할 수 있다.

worker node로 사용할 4개의 Server에서 command를 실행 후, 4개의 worker node가 swarm에 join 되었다면, manager node server에서 다음 명령어를 실행하자.

> - $docker node ls 

![image](https://user-images.githubusercontent.com/20153890/40535271-918f8dc8-6043-11e8-86da-4219bf45cfeb.png)

MANAGER STATUS column을 확인하면, Leader로 지정된 node를 확인할 수 있다. 이는 manager node임을 뜻한다.
그리고 4개의 worker node가 추가되어져 있는 것을 확인할 수 있을 것이다. 

 ### 3. deploying application services to the swarm
 
 ![image](https://user-images.githubusercontent.com/20153890/40592779-00e63ebc-625e-11e8-8983-8f39f9122083.png)

노란박스로 둘러 쌓여진 service들을 swarm에 deploy할 예정이다. 위의 service들은 MSA형태로 이루어진 서비스 들이고, API Gateway, 알림 Service, Service Discovery(eureka) 는 Spring Boot Project이고, 게시판 관리 Service는 NodeJS Service이다. 그림에선 볼 수 없지만, 게시판 관리 Service는 Spring Boot로 되어져 있는 API Gateway와 연결되기 위해 중간에 Spring-Cloud-Sidecar라는 또 하나의 Tomcat이 떠 있는 상태이다. 이렇게 하여 총 5개의 Service를 생성 할 것이다. 이 5개의 service는 docker image로 만들어 필자의 docker-hub에 올려 놓았고, 이 image들을 받아와 service로 deploy할 예쩡이다.

사실 Docker Swarm은 ingress라는 가상 network에 속해 있다. ingress에는 Routing Mesh라는 기능을 제공한다. 공식 문서에는 **All nodes participate in an ingress routing mesh. The routing mesh enables each node in the swarm to accept connections on published ports for any service running in the swarm, even if there’s no task running on the node. The routing mesh routes all incoming requests to published ports on available nodes to an active container.** 라고 설명한다.

service가 port를 열 경우, swarm에 속한 전체 node에 그 port가 열리게 되고, 어떠한 node의 port에 요청을 보내도 실행중인 container에 자동으로 전달해준다. 따라서 API Gateway가 굳이 필요하진 않다. 그러나 여기서는 service를 각 node들에 deploy하는게 목적이므로 신경쓰지 않고 진행하도록 하겠다.

이전 단계에서 swarm에 manager node와 worker node들을 추가하였으므로, service들을 swarm에 추가해보자.
service deploy는 manager node server에서 진행해야 한다. service는 위에서도 설명했다시피 docker image를 받아와 실행 할 것이다. 

정리해보면 API Gateway, 알림 Service(Spring Boot), Service Discovery(eureka), 게시판 관리 Service(NodeJS), Spring-Cloud-Sidecar 까지 총 5개의 service를 docker hub에서 image로 받아와 service를 swarm에 deploy하겠다.

### 4. managing the swarm once you have everything running
