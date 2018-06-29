Docker Swarm Tutorial
=====================
*by s.m.Lee*

![image](https://user-images.githubusercontent.com/20153890/40535846-41728da2-6045-11e8-87c7-e301beac3c46.png)

Orchestration과 Docker Swarm에 대해 알아보았고, Component 또한 어느 정도 이해가 되었을 것이라 생각한다.
그럼 이제 Swarm을 구성하여 보고, service를 deploy 해보자.

# Swarm Tutorial #

그럼 이제 docker cli를 통해 swarm을 만들고, swarm에 service를 deploy 해보자.

*공식 문서를 참고 하였고, 조금 다른 방향으로 진행해 보겠다*

**NOTE**

진행 순서는 아래와 같다.
##
 1. intializing a cluster of Docker Engines in swarm mode
 2. adding nodes to the swarm
 3. deploying application services to the swarm
##

이 튜토리얼을 진행하기 위해서는 

### 1. 5개의 linux host

   - 하나의 manager node와, 4개의 worker node로 구성할 것이다.

### 2. Docker Engine 1.12 or later installed 

   - install Docker Engine on Linux machines
   - 리눅스 기반의 OS를 사용 중이면, 공식 document의 install instruction을 보면 된다.
     이미 docker 가 설치되어 있다면 skip 하자

### 3. the IP address of the manager machine

   - IP address는 고정 ip가 할당되어 있어야 한다.
   - swarm에 속한 모든 node는 IP에 의해 manager에 connect 되어야 한다.

### 4. open ports between the hosts

   - TCP port 2377 - for cluster management communications
   - TCP and UDP port 7946 - for communication among nodes
   - UDP port 4789 for overlay network traffic



## Swarm

공식 문서에는 docker-machine을 이용해서 manager node를 실행시키고 ssh로 machine에 접속하라는 형태로 쓰여 있는데
linux host라면 docker-machine 구성없이 진행해도 된다.

### 1. intializing a cluster of Docker Engines in swarm mode

> - $docker swarm init --advertise-addr [MANAGER-IP]

manager node로 사용할 linux host server에 위 command를 실행한다. [MANAGER-IP] 부분에 manager node로 사용할 IP(를 입력하면 된다.

![image](https://user-images.githubusercontent.com/20153890/40534751-0bfa467c-6042-11e8-80cb-4ad6b612a9ec.png)

--advertise-addr flag는 manager node가 자신의 address를 192.168.10.174로 게시하도록 한다.

실행 결과, 중간에 새로운 node가 swarm에 join할수 있는 command(token을 포함한)를 포함하고 있다. 또 다른 node는 manager나 worker로 swarm에 join 할 수 있다. 여기서는 4개의 node들이 worker node로 swarm에 join할 것이므로 중간에 나와있는 **docker swarm join --token ~~ command**를 worker node로 지정할 server에서 실행시키면 된다.

### 2. adding nodes to the swarm

여기서는 4개의 worker node를 사용할 것이므로 worker로 사용 예정 중인 linux host server 4개에 각각 docker swarm join --token ~~ command를 실행하면 된다.
swarm에 join시킬 command를 잊었다면, $docker swarm join-token (manager or worker)를 manager node server에서 실행하면 확인할 수 있다.

worker node로 사용할 4개의 Server에서 command를 실행 후, 4개의 worker node가 swarm에 join 되었다면, manager node server에서 다음 명령어를 실행하자.


```bash
[sangmin@was ~]$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
vcjedw7k4op88iyl7hfvn2e9p     alarm               Ready               Active                                  18.03.1-ce
e4q1vt62gdrj32ak5k0k7vfzc     apitest             Ready               Active                                  18.03.0-ce
wngppkcnkviiqludqz7d1kdg6     boardapi            Ready               Active                                  18.03.1-ce
o4oinwydtft38p8o4dv2bhco1     eureka              Ready               Active                                  18.03.1-ce
peqjhis6oskrtb9oftne2dx49 *   was                 Ready               Active              Leader              18.03.1-ce
```

![image](https://user-images.githubusercontent.com/20153890/40535271-918f8dc8-6043-11e8-86da-4219bf45cfeb.png)

MANAGER STATUS column을 확인하면, Leader로 지정된 node를 확인할 수 있다. 이는 manager node임을 뜻한다.
그리고 4개의 worker node가 추가되어져 있는 것을 확인할 수 있을 것이다. 

 ### 3. deploying application services to the swarm
 
 ![image](https://user-images.githubusercontent.com/20153890/40592779-00e63ebc-625e-11e8-8983-8f39f9122083.png)


박스로 둘러 쌓여진 service들을 swarm에 deploy할 예정이다. 위의 service들은 MSA 형태로 이루어진 서비스 들이고, API Gateway, 알림 Service, Service Discovery(eureka) 는 Spring Boot Project이고, 게시판 관리 Service는 NodeJS Service이다. 그림에선 볼 수 없지만, 게시판 관리 Service는 Spring Boot로 되어 있는 API Gateway와 연결되기 위해 중간에 Spring-Cloud-Sidecar라는 또 하나의 Tomcat이 떠 있는 상태이다.

이렇게 하여 총 5개의 Service를 생성 할 것이다. 이 5개의 service는 docker image로 만들어 필자의 docker-hub에 올려 놓았고, 이 image들을 받아와 service로 deploy할 예정이다.

사실 Docker Swarm은 ingress라는 가상 network에 속해 있다. ingress에는 Routing Mesh라는 기능을 제공한다. 공식 문서에는 **All nodes participate in an ingress routing mesh. The routing mesh enables each node in the swarm to accept connections on published ports for any service running in the swarm, even if there’s no task running on the node. The routing mesh routes all incoming requests to published ports on available nodes to an active container.** 라고 설명한다.

service가 port를 열 경우, swarm에 속한 전체 node에 그 port가 열리게 되고, 어떠한 node의 port에 요청을 보내도 실행 중인 container에 자동으로 전달해준다. 따라서 API Gateway가 굳이 필요하진 않다. 그러나 여기서는 service를 각 node 들에 deploy 하는 게 목적이므로 신경 쓰지 않고 진행하도록 하겠다.

이전 단계에서 swarm에 manager node와 worker node들을 추가하였으므로, service들을 swarm에 추가해보자.
service deploy는 manager node server에서 진행해야 한다. service는 위에서도 설명했다시피 docker image를 받아와 실행할 것이다. 

**Service description**

Service Name | DockerId | Image | Port 
------------|-----|------------ | ---------------
api-gateway | phantasmicmeans | api-gateway | 4000
eureka | phantasmicmeans | eureka_server | 8761
alarm-service | phantasmicmeans | alarm-service | 8763
board-service | phantasmicmeans | board-service | 3000
board-service-sidecar | phantasmicmeans | board-service-sidecar | 8766

정리해보면 API Gateway, 알림 Service(Spring Boot API Server), Service Discovery(eureka), 게시판 관리 Service(NodeJS API Server), Spring-Cloud-Sidecar 까지 총 5개의 service를 docker hub에서 image로 받아와 service를 swarm에 deploy하겠다.

```bash
$docker service create \
    --name eureka \  # 생성 될 Service의 이름을 eureka로 명시한다.
    --constraint 'node.hostname == eureka' \ # service를 eureka라는 이름을 가진 host server에서만 생성한다. 
    --replicas 3 \ # service scaling을 통해 service를 3개로 늘린다. 
    --publish 8761 \ # publishedport를 8761로 지정한다. 
    phantasmicmeans/eureka_server # service 생성에 사용할 Docker Image이다. 
```

위의 명령어는 eureka service를 생성하고, node.hostname == eureka인 node에 service를 deploy 하는 command이다.
여기서는 constraint option을 주어 하나의 노드에 특정 service만 deploy 되도록 지정하였으나, 여러 node들에 service가 deploy되길 원한다면 constraint flag를 지워주면 된다. 위의 service description을 참고하여 service를 deploy 해도 되고, 필자가 작성해 놓은 script를 이용해도 된다.
manager node server에서 위 command를 실행하면 된다.

> - $./DeployList

모든 service가 정상적으로 각 worker node에 deploy 되었다면 manager node server에서 다음과 같은 command를 실행해 보자.

```bash
[sangmin@was ~]$ docker service ls
ID                  NAME                    MODE                REPLICAS            IMAGE                                          PORTS
kmi3jspm5eyx        alarm-service           replicated          3/3                 phantasmicmeans/alarm-service:latest           *:8763->8763/tcp
2v74fxq8wak0        api-gateway             replicated          3/3                 phantasmicmeans/api-gateway:latest             *:4000->4000/tcp
y5dqg0sbnbmn        board-service           replicated          3/3                 phantasmicmeans/board-service:latest           *:3000->3000/tcp
r19p7re9exyb        board-service-sidecar   replicated          3/3                 phantasmicmeans/board-service-sidecar:latest   *:8766->8766/tcp
smw41buddyiz        eureka                  replicated          3/3                 phantasmicmeans/eureka_server:latest           *:8761->8761/tcp
```

다음과 같이 5개의 service가 3개의 replica를 가지고 잘 생성된 것을 볼 수 있다.

그럼 이제 지정해준 worker node에 제대로 deploy 되었는지 확인해보자

***eureka service***

```bash
[sangmin@was ~]$ docker service ps eureka
ID                  NAME                IMAGE                                  NODE                DESIRED STATE       CURRENT STATE        ERROR               PORTS
q238ke65sv88        eureka.1            phantasmicmeans/eureka_server:latest   eureka              Running             Running 3 days ago    
m0cgt4sjjbji        eureka.2            phantasmicmeans/eureka_server:latest   eureka              Running             Running 3 days ago    
r3bm3mrzb7lx        eureka.3            phantasmicmeans/eureka_server:latest   eureka              Running             Running 3 days ago    
```

***api-gateway service***

```bash
[sangmin@was ~]$ docker service ps api-gateway
ID                  NAME                IMAGE                                NODE                DESIRED STATE       CURRENT STATE        ERROR               PORTS
5gg0e9qgkvvq        api-gateway.1       phantasmicmeans/api-gateway:latest   apitest             Running             Running 3 days ago      
q2b274yg6fu2        api-gateway.2       phantasmicmeans/api-gateway:latest   apitest             Running             Running 3 days ago      
xxn0v75c74ch        api-gateway.3       phantasmicmeans/api-gateway:latest   apitest             Running             Running 3 days ago 

```

eureka service와 api-gateway service가 3개의 replica를 가지고 각각 eureka라는 이름을 가진 node와 apitest라는 이름을 가진 node로 deploy 되어 있는 것을 확인할 수 있을 것이다.

worker node로 사용하고 있는 server에서 cotainer의 상태를 확인하여도 된다. eureka service로 활용하고 있는 server에서 container의 상태를 확인하여 보겠다. 다음 command를 eureka service를 deploy한 worker node server에서 실행한다.

```bash
[sangmin@eureka ~]$ docker ps
CONTAINER ID        IMAGE                                  COMMAND                  CREATED             STATUS              PORTS               NAMES
3c15340df6f8        phantasmicmeans/eureka_server:latest   "java -Djava.securit…"   3 days ago          Up 3 days                               eureka.2.m0cgt4sjjbjicmj35xztcauzb
36c9676f6ef0        phantasmicmeans/eureka_server:latest   "java -Djava.securit…"   3 days ago          Up 3 days                               eureka.1.q238ke65sv880sxqn5z1hyqag
c302798d830f        phantasmicmeans/eureka_server:latest   "java -Djava.securit…"   3 days ago          Up 3 days                               eureka.3.r3bm3mrzb7lx1f5m3az8grn3b
```

위처럼 3개의 eureka replicated 되어진 container가 실행되고 있는 것을 볼 수 있을 것이다.

***Conclusion***

지금까지 docker-swarm을 활용해 여러 service를 container에 적용해 보았고, 이 cluster에 node들을 포함시키고 각 node 들에 여러 service를 deploy 해 보았다.

docker-swarm은 기능적인 측면이나 활용도 측면에서 kubernetes에 비해 다소 낮게 평가 된다. 그러나 사용 방법도 쉽고 구축 비용도 적으므로 규모가 크지 않은 container를 관리하는 데에는 docker-swarm이 가장 잘 어울리지 않을까 라는 생각을 하게 된다.












