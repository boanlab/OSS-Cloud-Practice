# 오픈소스SW(클라우드)

# 04. 도커 네트워크

> (본 실습은 Ubuntu 24.04 VM 환경을 기준으로 작성되었습니다.)

---

## Step 1. 기본 네트워크 구조 및 인터페이스 확인
도커 컨테이너가 생성되었을 때 기본적으로 할당되는 가상 인터페이스와 브리지를 확인하고 매핑 관계를 대조해 봅니다.
```bash
# 0. 실습용 백그라운드 컨테이너 실행
$ docker run -d --name test-net alpine sleep 10000
```
![figure1](./images/figure1.png)

```bash
# 1. 컨테이너 내부의 eth0 인터페이스 확인
docker exec [container_name_or_id] ip addr show eth0
```
![figure2](./images/figure2.png)

```bash
# 2. 호스트의 veth 인터페이스 목록 확인 및 브리지(docker0) 구성 상태 확인
ip link show | grep veth
sudo apt install bridge-utils
sudo brctl show docker0
```
![figure3](./images/figure3.png)

![figure4](./images/figure4.png)

```bash
# 3. veth 페어 인덱스 대조하기 (컨테이너 eth0 호스트 veth 매칭)
# 컨테이너 내부 eth0의 iflink 인덱스 확인 (예: 47)
docker exec [container_name_or_id] cat /sys/class/net/eth0/iflink

![figure5](./images/figure5.png)

# 호스트에서 해당 인덱스를 가진 veth 찾기
grep -l 47 /sys/class/net/veth*/ifindex
```
![figure6](./images/figure6.png)

| 명령어/옵션 | 설명 |
|---|---|
| `ip addr show` | 네트워크 인터페이스와 할당된 IP 주소를 확인합니다. |
| `ip link show` | 활성화된 모든 링크 계층(가상/물리) 인터페이스 상태를 출력합니다. |
| `brctl show` | 리눅스 브리지의 구성 상태와 연결된 포트(veth) 목록을 출력합니다. |
| `docker network inspect` | 특정 도커 네트워크의 상세 정보(서브넷, IPAM, 컨테이너 등)를 조회합니다. |
---

## Step 2. 네트워크 네임스페이스 격리 원리 분석
도커가 어떻게 프로세스의 네트워크 스택을 격리하는지 리눅스 커널 관점에서 직접 확인합니다.
```bash
# 1. 테스트용 alpine 컨테이너 프로세스(PID) 조회
ps aux | grep 'sleep 10000'
```
![figure7](./images/figure7.png)

```bash
# 2. 해당 프로세스(PID)의 네임스페이스 목록 확인
# PID가 12345라고 가정
sudo ls /proc/12345/ns/
```
![figure8](./images/figure8.png)

```bash
# 3. nsenter를 사용하여 네임스페이스 내부 인터페이스 직접 조회
sudo nsenter -t 12345 -n ip addr show
```
![figure9](./images/figure9.png)

| 명령어/옵션 | 설명 |
|---|---|
| `ls /proc/[PID]/ns/` | 리눅스 커널이 해당 프로세스에 할당한 네임스페이스(net, ipc, pid 등) 파일들을 확인합니다. |
| `nsenter -t [PID] -n` | 특정 프로세스의 네트워크 격리 공간(Network Namespace) 내부로 직접 진입하여 명령을 실행합니다. |
---

## Step 3. 도커 네트워크 통신 제어 (iptables & NAT)
컨테이너 외부 통신 및 포트 포워딩을 제어하는 리눅스 커널의 방화벽/NAT 규칙을 확인합니다.
```bash
# 1. 현재 적용된 NAT 테이블 규칙 목록 확인
sudo iptables -t nat -L
```
![figure10](./images/figure10.png)

```bash
# 2. 외부 포트 8081을 내부 컨테이너 80번 포트로 전달 (DNAT 예시)
sudo iptables -t nat -A PREROUTING -p tcp --dport 8081 -j DNAT --to-destination 172.17.0.2:80
```
![figure11](./images/figure11.png)

```bash
# 3. iptables의 FORWARD 체인 기본 정책 확인 및 허용(ACCEPT)으로 변경
sudo iptables -L | grep FORWARD
sudo iptables --policy FORWARD ACCEPT
```
![figure12](./images/figure12.png)

| 옵션/명령어 | 설명 |
|---|---|
| `iptables -t nat -L` | 네트워크 주소 변환(NAT)에 관련된 방화벽 룰 리스트를 출력합니다. |
| `DNAT` | Destination NAT. 외부에서 들어오는 트래픽의 목적지 IP/포트를 컨테이너로 포워딩합니다. |
| `MASQUERADE` | 동적 IP 환경에서 컨테이너의 사설 IP를 호스트의 공인 IP로 변환(SNAT)하여 외부 접속을 돕습니다. |
| `--policy FORWARD ACCEPT`| 브리지를 통과해 네임스페이스 간 전달되는 패킷의 포워딩을 전면 허용합니다. |
---

## Step 4. 유사 컨테이너 네트워크 수동 구축 (Core Lab)

리눅스 기본 명령어만 사용하여 두 개의 격리된 공간을 만들고 가상 브리지로 통신시키는 전체 
과정을 실습합니다.

---
```bash
# 1. ns1, ns2 네트워크 네임스페이스 생성 및 확인
sudo ip netns add ns1
sudo ip netns add ns2
ip netns list
```
![figure13](./images/figure13.png)

```bash
# 2. 호스트에 br0라는 가상 브리지 생성 및 활성화
sudo ip link add name br0 type bridge
sudo ip link set br0 up
```
![figure14](./images/figure14.png)

```bash
# 3. veth pair 생성 및 브리지 연결 활성화
sudo ip link add veth1 type veth peer name veth-ns1
sudo ip link add veth2 type veth peer name veth-ns2
sudo brctl addif br0 veth-ns1
sudo brctl addif br0 veth-ns2
sudo ip link set dev veth-ns1 up
sudo ip link set dev veth-ns2 up
```
![figure15](./images/figure15.png)

```bash
# 4. 네임스페이스 내부 IP 할당 및 인터페이스(lo 포함) 활성화
sudo ip link set veth1 netns ns1
sudo ip netns exec ns1 ip addr add 192.168.1.2/24 dev veth1
sudo ip netns exec ns1 ip link set dev veth1 up
sudo ip netns exec ns1 ip link set lo up

sudo ip link set veth2 netns ns2
sudo ip netns exec ns2 ip addr add 192.168.1.3/24 dev veth2
sudo ip netns exec ns2 ip link set dev veth2 up
sudo ip netns exec ns2 ip link set lo up
```
![figure16](./images/figure16.png)

```bash
# 5. 격리된 네임스페이스 간 상호 통신(Ping) 테스트
sudo ip netns exec ns1 ping -c 1 192.168.1.3
sudo ip netns exec ns2 ping -c 1 192.168.1.2
```
![figure17](./images/figure17.png)

---
## Step 5. 포트 퍼블리싱 실습 (EXPOSE vs PUBLISH)
격리된 컨테이너의 포트를 외부로 개방(바인딩)하는 방법입니다.

---
```bash
# 1. [Host] 호스트 네트워크 스택 직접 공유 (격리 해제)
docker run -d --net host --name my-ubuntu ubuntu tail -f /dev/null
```
![figure18](./images/figure18.png)

```bash
# 2. [None] 완벽한 네트워크 격리 (Loopback만 존재)
docker run --rm -d --net none --name no-net-nginx nginx
```
![figure19](./images/figure19.png)

```bash
# 3. [Overlay] 다중 호스트 환경의 분산 네트워크 생성 (Swarm 매니저 기준)
docker swarm init
docker network create -d overlay my-overlay
docker service create --name my-web --replicas 3 --network my-overlay -p 80:80 nginx
```
![figure20](./images/figure20.png)

```bash
# 4. [Ipvlan/Macvlan] 물리적 네트워크 인터페이스 기반 통신 생성
docker network create -d macvlan \
  --subnet=192.168.10.0/24 \
  --gateway=192.168.10.1 \
  -o parent=ens3 \
  macvlan-net

docker run -it --rm --network macvlan-net alpine ip addr show eth0
```
![figure21](./images/figure21.png)

---

# 05. 도커 볼륨과 로그 관리

> (본 실습은 Ubuntu 24.04 VM 환경을 기준으로 작성되었습니다.)

---

## Step 1. 도커 볼륨 생성 및 유형별 마운트
컨테이너의 생명주기(Lifecycle)와 무관하게 데이터를 안전하게 보존하기 위한 볼륨(Volume)을 생성하고 확인합니다.

```bash
# 1. 'db'라는 이름의 Named Volume 생성
docker volume create --name db
```
![figure22](./images/figure22.png)

```bash

# 2. 생성된 볼륨 상세 정보 조회 (호스트의 실제 저장 경로 확인)
# 기본 저장 경로는 /var/lib/docker/volumes/ 입니다.
docker volume inspect db
```
![figure23](./images/figure23.png)

```bash
# 3. Anonymous Volume 생성 (이름 없이 경로만 지정)
docker run -d --name nginx-anon -v /usr/share/nginx/html nginx
```
![figure24](./images/figure24.png)

```bash
# 4. 전체 볼륨 목록 확인 (Named와 Anonymous 볼륨 비교)
# Anonymous 볼륨은 64자 길이의 해시값으로 표시됩니다.
docker volume ls
```
![figure25](./images/figure25.png)

| 명령어 | 설명 |
|:---|:---|
| `docker volume create` | 명시적인 이름으로 새로운 볼륨을 생성하여 관리 용이성을 확보합니다. |
| `docker volume ls` | 호스트 내 생성된 모든 도커 볼륨 목록을 조회하여 확인합니다. |
| `docker volume inspect` | 볼륨의 호스트 마운트 경로 등 상세 정보를 확인합니다. |
---
## Step 2. Host Volume 및 읽기 전용(:ro) 마운트
호스트 운영체제의 특정 디렉토리를 직접 연결(Bind Mount)하고, 보안을 위해 읽기 전용으로 설정하는 방법을 실습합니다.

---
```bash
# 1. 호스트에 테스트용 디렉토리 및 인덱스 파일 생성
sudo mkdir -p /opt/html_files
echo "Hello, World!" | sudo tee /opt/html_files/index.html
```
![figure26](./images/figure26.png)

```bash
# 2. 호스트 디렉토리를 컨테이너에 마운트하여 Nginx 실행 (-v 호스트경로:컨테이너경로)
docker run -d --name nginx-host-vol -v /opt/html_files:/usr/share/nginx/html -p 8082:80 nginx
curl localhost:8082

```
![figure27](./images/figure27.png)

```bash
# 3. 데이터 변조 방지를 위한 읽기 전용(:ro) 마운트 실행
docker volume create web-volume
docker run -d --name nginx-ro -v web-volume:/usr/share/nginx/html:ro nginx
```
![figure28](./images/figure28.png)

```bash
# 4. 쓰기 작업 시도 및 실패 확인 (Read-only file system 에러 발생)
docker exec nginx-ro touch /usr/share/nginx/html/test.txt
```
![figure29](./images/figure29.png)

## Step 3. 볼륨 컨테이너 공유 (--volumes-from)
특정 컨테이너를 데이터 저장소 전용으로 지정하고, 다른 컨테이너들이 이 마운트 설정을 상속받아 데이터를 공유하는 패턴입니다.

---
```bash
# 1. 데이터를 저장할 전용 컨테이너(my-volume) 생성
docker run -d -it --name my-volume -v /opt/html_files:/usr/share/nginx/html ubuntu:focal
```
![figure30](./images/figure30.png)

```bash
# 2. --volumes-from 옵션으로 마운트 정보 상속받아 Nginx 실행
docker run -d --name nginx-shared --volumes-from my-volume nginx
```
![figure31](./images/figure31.png)

## Step 4. 상태 유지형(Stateful) 애플리케이션: MySQL 데이터 영속성 검증
컨테이너가 삭제되어도 데이터베이스 내부의 정보가 안전하게 유지되는지 확인합니다.

---

```bash
# 1. 전용 볼륨 생성 및 MySQL 컨테이너 실행
docker volume create db-volume
docker run --name my-mysql -d -e MYSQL_ROOT_PASSWORD=password -v db-volume:/var/lib/mysql -p 3306:3306 mysql:5.7
```
![figure32](./images/figure32.png)

```bash
# 2. 컨테이너 내부 쉘로 접속하여 테스트 DB 생성 (Interactive Mode)
docker exec -it my-mysql mysql -u root -p
# 비밀번호 입력 후 MySQL 프롬프트에서:
mysql> CREATE DATABASE mydb;
mysql> SHOW DATABASES;
mysql> exit
```
![figure33](./images/figure33.png)

```bash
# 3. 컨테이너 강제 종료 및 재시작
docker stop my-mysql
docker start my-mysql
```
![figure34](./images/figure34.png)

```bash
# 4. 재접속하여 DB 유지 여부 검증
docker exec -it my-mysql mysql -u root -p
# 로그인 후:
mysql> SHOW DATABASES;
# 기존에 생성한 'mydb'가 그대로 유지되고 있는지 확인합니다.
```
![figure35](./images/figure35.png)

## Step 5. [보안 심화] LUKS 기반 도커 볼륨 암호화
리눅스 표준 디스크 암호화(LUKS)를 활용하여 호스트 수준에서 데이터를 암호화하고 도커에 마운트합니다.

```bash
# 1. 1GB 더미 파일 생성 및 루프백 디바이스 연결
sudo dd if=/dev/zero of=secret_vol bs=1M count=1024
sudo ln -s $(sudo losetup -f --show secret_vol) /dev/loop_boan
```
![figure36](./images/figure36.png)

```bash
# 2. LUKS 포맷(초기화) 및 암호화된 볼륨 매핑 열기
# 포맷 확인 시 반드시 대문자 YES 입력 후, 사용할 패스워드를 지정합니다.
sudo cryptsetup luksFormat /dev/loop_boan
sudo cryptsetup luksOpen /dev/loop_boan encrypted_vol
```
![figure37](./images/figure37.png)
![figure38](./images/figure38.png)

```bash
# 3. 매핑된 장치에 파일시스템(ext4) 생성 및 디렉토리 마운트
sudo mkfs.ext4 /dev/mapper/encrypted_vol
sudo mkdir -p /mnt/secure_data
sudo mount /dev/mapper/encrypted_vol /mnt/secure_data
```
![figure39](./images/figure39.png)

```bash
# 4. 암호화된 보안 디렉토리를 컨테이너 볼륨으로 연결하여 실행
docker run -d --name secure-db -v /mnt/secure_data:/var/lib/mysql mysql:5.7
```
![figure40](./images/figure40.png)

```bash
# 5. [실습 종료 후 정리] 마운트 및 암호화 채널 안전하게 해제
docker rm -f secure-db
sudo umount /mnt/secure_data
sudo cryptsetup luksClose encrypted_vol
sudo losetup -d /dev/loop_boan
sudo rm /dev/loop_boan
```
![figure41](./images/figure41.png)

## Step 6. 컨테이너 로그 관리 및 용량 제한 (Log Rotation)
표준 입출력(STDOUT/STDERR)으로 발생하는 로그를 확인하고, 디스크 고갈을 막기 위한 로테이션 정책을 적용합니다.

---

```bash
# 1. 실시간 로그 스트림 확인 및 타임스탬프 표시
docker logs -f my-mysql
docker logs --tail 10 -t my-mysql
```
![figure42](./images/figure42.png)
![figure43](./images/figure43.png)

```bash
# 2. 디스크 용량 관리를 위한 로그 로테이션 옵션 적용하여 컨테이너 실행
# 파일당 최대 3MB, 보관 파일 최대 5개로 제한합니다.
docker run -d \
  --log-driver=json-file \
  --log-opt max-size=3m \
  --log-opt max-file=5 \
  nginx
```
![figure44](./images/figure44.png)

| 명령어/옵션 | 설명 |
|:---|:---|
| `docker logs -f` | 생성되는 로그를 실시간으로 계속 출력(Follow)합니다. |
| `docker logs --tail n` | 마지막 n줄의 로그만 확인하여 스크롤 낭비를 방지합니다. |
| `--log-opt max-size` | 단일 로그 파일의 최대 허용 크기를 설정합니다 (예: 10m, 1g). |
| `--log-opt max-file` | 보관할 최대 로그 파일 개수를 설정하여 로테이션을 적용합니다. |


