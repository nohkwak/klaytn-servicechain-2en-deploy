# 2 EN on Docker 
[Klaytn Service Chain](https://ko.docs.klaytn.com/node/service-chain) 노드 4개를 Baobab EN 노드 2개와 연결하여 HA 구성하기 위한 스크립트 및 설정 파일



## Prerequisites
Followings are required.

1. [Docker](https://docs.docker.com/get-docker/)
2. [Docker-compose](https://docs.docker.com/compose/install/)


## EN 2개 노드 구성
### 작업 디렉토리 및 파일 생성
```
$ mkdir ~/ken2
$ cd ~/ken2
$ curl -X GET https://packages.klaytn.net/baobab/genesis.json -o ./genesis.json
$ mv docker-compose.yaml  ./
```

### Docker 실행 및 확인
```
$ docker-compose up

$ docker ps
   Name       Command   State                                                             Ports                                                          
---------------------------------------------------------------------------------------------------------------------------------------------------------
ken2_EN-0_1   /bin/sh   Up      0.0.0.0:32321->32323/tcp, 32323/udp, 0.0.0.0:50501->50505/tcp, 0.0.0.0:61001->61001/tcp, 0.0.0.0:8551->8551/tcp, 8552/tcp
ken2_EN-1_1   /bin/sh   Up      0.0.0.0:32322->32323/tcp, 32323/udp, 0.0.0.0:50502->50505/tcp, 0.0.0.0:61002->61001/tcp, 0.0.0.0:8552->8551/tcp, 8552/tcp
```

### 접속 방법
```
$ sudo docker exec -it ken2_EN-0_1 bash
$ sudo docker exec -it ken2_EN-1_1 bash
```


## EN 노드 초기화 
```
$ mkdir /data
$ ken --datadir /data init klaytn/genesis.json
```


### 설정 업데이트 
/klaytn-docker-pkg/conf/kend.conf 정보를 업데이트한다. 
```
...
NETWORK="baobab"
...
SC_MAIN_BRIDGE=1
...
DATA_DIR=/data
...
```

### 실행
```
$ kend start
Starting kend: OK
```


### 실행 상태 확인 방법
klay.blockNumber로 블록 생성 상태를 확인할 수 있으며, 이 숫자가 0이 아니면 노드가 제대로 동작하는 것이다.
```
$ ken attach --datadir /data

> klay.blockNumber
54726

> klay.blockNumber
108795

> klay.blockNumber
111536
```


### EN의 KNI 정보 조회
```
$ ken attach --datadir /data

> mainbridge.nodeInfo.kni
"kni://...@[::]:50505?discport=0"
```



## SCN 설정 정보 업데이트
### KNI 정보 업데이트 
EN이 아니라 SCN의 /data/main-bridges.json 파일에 위의 KNI 정보와 IP 및 Port 정보를 업데이트 한다. 
```
["kni://...@198.168.0.16:50501?discport=0"]
```

### conf 파일 업데이트 
SCN의 klaytn/conf/kscnd.conf 파일을 아래와 같이 업데이트 한다. 
```
...
SC_SUB_BRIDGE=1
...
SC_PARENT_CHAIN_ID=1001
...
SC_TX_PERIOD=10
...
```

### SCN 재부팅 
```
$ kscnd stop
Shutting down kscnd: Killed
$ kscnd start
Starting kscnd: OK
```
kscnd stop이 제대로 동작 안 할 경우 ps, kill로 프로세스를 종료시킴 
```
$ ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    7 root      0:00 bash
   38 root      1h20 /klaytn-docker-pkg/bin/kscn --nodiscover --metrics --prometheus --multichannel --rpc --rpcapi klay,subbridge --rpcport 8551 --rpcaddr 0.0.0.0 --rpccorsdomain *
   53 root      2:03 kscn attach --datadir /data
   70 root      0:00 bash
  174 root      0:00 vi testkey1
  196 root      0:00 bash
  217 root      0:00 ps
$ kill -9 38
```


### EN 연결 여부 확인 
subbridge.peers.length를 검사하여 SCN이 EN에 연결되어 있는지 확인한다. 
```
$ kscn attach --datadir ~/data
> subbridge.peers.length
1
```

### 바오밥 부모 계정에 KLAY 충전
[Baobab Wallet Faucet](https://baobab.wallet.klaytn.com/faucet)에서 KLAY를 받은 후 subbridge.parentOperator 계정에 전송한다.
```
$ kscn attach --datadir ~/data
> subbridge.parentOperator
"0x......"
```

### 앵커링 시작 
```
$ kscn attach --datadir ~/data
> subbridge.anchoring(true)
true
> subbridge.latestAnchoredBlockNumber
100
```

