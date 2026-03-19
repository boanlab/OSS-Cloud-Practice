# 오픈소스SW(클라우드)

# 02. 도커 이미지와 컨테이너

> (본 실습은 Ubuntu 24.04 VM 환경을 기준으로 작성되었습니다.)

---

## Step 1. 도커 엔진 설치
가장 먼저 Ubuntu 24.04 환경에 도커를 설치하고 권한을 설정합니다.
```bash
# 1. 패키지 인덱스 업데이트 및 필수 패키지 설치
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
```
<img width="1000" height="451" alt="image" src="https://github.com/user-attachments/assets/fda4ab00-bbc1-4d88-89cb-8569316b8bbb" />
<img width="1000" height="451" alt="image" src="https://github.com/user-attachments/assets/40d44eef-5e4a-4890-81be-b3f857e16279" />

---

```bash
# 2. 기존 구버전 도커 패키지 제거 (기존에 도커가 설치되어 있었다면 꼭 제거할 것)
sudo apt-get remove -y docker docker-engine docker.io containerd runc || true
```
<img width="1000" height="143" alt="image" src="https://github.com/user-attachments/assets/d3624711-9970-472c-9470-13b6c8030e61" />

---

```bash
# 3. Docker 공식 GPG 키 등록 및 저장소 추가
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
<img width="1000" height="143" alt="image" src="https://github.com/user-attachments/assets/60d3ccb9-2f08-4774-b778-438996612dad" />

---

```bash
# 4. 도커 엔진 설치
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
<img width="1000" height="186" alt="image" src="https://github.com/user-attachments/assets/48c21624-4213-41b4-acc0-0597fc6f0cef" />
<img width="1000" height="454" alt="image" src="https://github.com/user-attachments/assets/eb926801-3dc6-4b8b-a31c-b93352e35b20" />

---

```bash
# 5. 설치 확인 
docker --version
sudo docker run --rm hello-world
```
<img width="1000" height="46" alt="image" src="https://github.com/user-attachments/assets/534175ed-f341-47e9-9b5c-8ee239106bda" />
<img width="1000" height="453" alt="image" src="https://github.com/user-attachments/assets/cfacf4a2-3532-4b40-a5ce-39c27be2b84b" />

---

```bash
# 6. 현재 사용자에게 docker 그룹 권한 부여 (이후 sudo 생략 가능)
sudo usermod -aG docker $USER
newgrp docker
```
<img width="1000" height="68" alt="image" src="https://github.com/user-attachments/assets/365c9ee2-2cfb-4917-bef4-e54b850f4f09" />

---

```bash
# 7. 권한 적용 후 재검증
docker run --rm hello-world
```
<img width="1000" height="427" alt="image" src="https://github.com/user-attachments/assets/ea6a3439-11e0-4694-adcb-8af109fbecc6" />

---

### 컨테이너 생성 및 실행 (`docker run`)
| 옵션 | 설명 |
| :--- | :--- |
| `-i` | 표준 입력(stdin)을 활성화하며, 터미널 연결 없이도 입력을 유지합니다. |
| `-t` | 가상 터미널(TTY)을 할당하여 텍스트 기반 상호작용을 지원합니다. |
| `-d` | 컨테이너를 백그라운드에서 실행하고 ID를 출력합니다. |
| `--rm` | 컨테이너 종료 시 관련 파일 시스템을 자동 삭제합니다. |
| `--name` | 컨테이너의 이름을 지정합니다. |
| `-p` | 호스트와 컨테이너의 포트를 연결합니다. |
| `-v` | 호스트 파일 시스템 경로를 마운트하여 데이터를 공유합니다. |
| `-e` | 단일 환경변수를 `키=값` 형식으로 설정합니다. |
| `--env-file` | 환경변수가 정의된 파일(`.env`)을 로드하여 일괄 적용합니다. |

## Step 2. 이미지 다운로드 및 레지스트리 관리 (PDF 19~21, 23쪽)
도커 허브에 로그인하고 실습에 사용할 이미지를 준비합니다.

https://hub.docker.com/

```bash
# 1. 도커 허브 로그인 
# (터미널에서 바로 비밀번호를 치려면 -u 옵션과 함께 본인 아이디를 입력하세요)
docker login -u 본인 Username

```
<img width="1000" height="141" alt="image" src="https://github.com/user-attachments/assets/7e9af7fd-4beb-47c3-8d52-65c8f70a11b2" />

---

```bash
# 2. 특정 버전의 Nginx 및 실습용 boanlab 이미지 다운로드
docker pull nginx:1.21
docker pull boanlab/hello-world:latest
```
<img width="1000" height="443" alt="image" src="https://github.com/user-attachments/assets/b4c9c1f7-6d84-40f4-a931-baa4b34ab784" />

---

```bash
# 3. 로컬에 저장된 이미지 목록 확인
docker images
docker images nginx # nginx 이미지만 조회
```
<img width="1000" height="137" alt="image" src="https://github.com/user-attachments/assets/7ba33191-a7c2-4dc3-a5ae-7bc7e7ad7d24" />
<img width="1000" height="204" alt="image" src="https://github.com/user-attachments/assets/06e39cdd-fc7a-4cae-926a-1d74fc0209ec" />

---

```bash
# 4. 이미지 필터링 및 포맷팅 조회
docker images | grep nginx #
docker images --filter "reference=nginx:*"
docker images --format "{{.Repository}}:{{.Tag}}"
```
<img width="1000" height="220" alt="image" src="https://github.com/user-attachments/assets/bd72a3f0-e3d7-40c3-9b65-640b089f8cc7" />

---

```bash
# 5. 이미지에 새로운 태그 부여 (버전 관리/별칭 생성)
# 💡 팁: 'myrepo' 부분을 본인의 실제 Docker Hub 아이디로 변경하세요!
docker tag nginx:1.21 myrepo/nginx:v1
```
```bash
# 6. 태그된 이미지를 레지스트리에 업로드
# 💡 팁: 위에서 본인의 아이디로 태그를 만들었다면, push 할 때도 권한 오류가 나지 않습니다.
docker push myrepo/nginx:v1
```
<img width="1000" height="54" alt="image" src="https://github.com/user-attachments/assets/c767089c-ba9a-486e-b139-69caf2ded7d6" />

---

```bash
# 7. 이미지 레이어 구조 및 히스토리 분석
docker history nginx:1.21
<img width="1000" height="323" alt="image" src="https://github.com/user-attachments/assets/168add22-7d0f-48ff-a905-ce0425a3df52" />

---

```
### 이미지 조회 (`docker images`)
| 옵션 | 설명 |
| :--- | :--- |
| `-q` | 이미지 ID만 간략하게 출력합니다. |
| `--filter` | 특정 조건(예: 미사용 이미지 등)으로 이미지를 검색합니다. |
| `--format` | 특정 컬럼만 출력하거나 포맷을 변경합니다. |

## Step 3. 웹 서버 컨테이너 구동 및 볼륨 마운트 (PDF 7~8, 18쪽)
준비된 이미지를 기반으로 컨테이너를 생성하고 호스트와 데이터를 공유합니다.
```bash

# 1. 호스트 디렉토리 및 테스트 파일 준비 (권한 부족 시 sudo 사용)
sudo mkdir -p /opt/hello
sudo sh -c 'echo "Hello from the host!" > /opt/hello/test.txt'
```
<img width="1000" height="102" alt="image" src="https://github.com/user-attachments/assets/7ed00883-3881-4d19-989e-e1f5df596cd8" />

---

```bash
# 2. 컨테이너 생성 (실행 준비 프로세스 확인용)
docker create --name web-ready nginx:1.21
```
<img width="1000" height="156" alt="image" src="https://github.com/user-attachments/assets/74b6a2c7-065d-47a7-9f87-9bcebe077112" />

---

```bash
# 3. Nginx 웹 서버 컨테이너 백그라운드 실행
# 💡 8080 포트 충돌을 피하기 위해 8081 포트를 사용합니다.
docker run -d --name web -p 8081:80 -v /opt/hello:/usr/share/nginx/html nginx:1.21
```
```bash
# 4. 웹 서버 정상 동작 검증 (터미널에 'Hello from the host!'가 출력되면 성공)
curl http://localhost:8081/test.txt
```
<img width="1000" height="170" alt="image" src="https://github.com/user-attachments/assets/5434d30e-a744-4bca-8aea-a23d40e80d79" />

---

```bash
# 5. [참고] 이미지 태깅 및 환경변수 주입 실습
docker tag nginx:1.21 my-app:latest

# 단일 환경변수 주입하여 실행
docker run --name env-test -e APP_ENV=prod -e DB_HOST=db-server -d my-app
# 환경변수 정상 주입 여부 확인 (APP_ENV=prod 출력됨)
docker exec env-test env | grep APP_ENV

# 파일(.env)을 통한 일괄 주입 실행
echo "API_KEY=secret123" > .env
docker run --name env-file-test --env-file .env -d my-app
# 환경변수 정상 주입 여부 확인 (API_KEY=secret123 출력됨)
docker exec env-file-test env | grep API_KEY

```
<img width="1004" height="360" alt="image" src="https://github.com/user-attachments/assets/9d17c1ec-ed16-4296-a798-3593c3ddeee8" />

---

## Step 4. 상태 모니터링 및 컨테이너 내부 제어 (PDF 9 ~ 11, 15 ~ 17쪽)
실행 중인 컨테이너의 상태를 점검하고 내부로 진입하여 작업을 수행합니다.
```bash

# 1. 컨테이너 목록 확인
docker ps
docker ps -a
docker ps -q
```
<img width="1017" height="290" alt="image" src="https://github.com/user-attachments/assets/ec013bc5-dce0-4a62-a250-f17bf750371d" />

```bash
# 2. 필터링 및 테이블 형태 포맷팅 출력
docker ps --filter "status=running" --format "table {{.ID}}\t{{.Image}}\t{{.Status}}"
```
![figure1](./images/figure1.png)
---

```bash
# 3. 컨테이너 상세 정보(JSON) 확인 및 특정 필드 추출
docker inspect web
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web
```
![figure2](./images/figure2.png)
![figure3](./images/figure3.png)

---

```bash
# 4. 리소스 사용량 실시간 확인 (종료하려면 Ctrl+C)
docker stats
```
![figure4](./images/figure4.png)

```bash
# 5. 실시간 로그 스트리밍 (종료하려면 Ctrl+C)
docker logs -f --tail 10 --timestamps web
```
![figure5](./images/figure5.png)

---

```bash

# 6. 호스트와 컨테이너 간 파일 복사
# 호스트 -> 컨테이너 (root가 생성한 파일이므로 sudo 필요)
sudo docker cp /opt/hello/test.txt web:/usr/share/nginx/html/test2.txt
# 컨테이너 -> 호스트 (로그 추출 등)
docker cp web:/var/log/nginx/access.log ./access.log

# 복사된 로그 파일 상태 및 크기 확인
ls -l ./access.log #
```
![figure6](./images/figure6.png)
---

```bash
# 7. 컨테이너 접속 (선택적 실행)
# 새로운 셸을 생성하여 내부 진입 (나올 때는 'exit' 입력)
docker exec -it web bash

# 메인 프로세스(PID 1)에 연결 (주의: Ctrl+C 입력 시 컨테이너가 종료됨)
docker attach web
```
![figure7](./images/figure7.png)
---

### 로그 확인 (`docker logs`)
| 옵션 | 설명 |
| :--- | :--- |
| `-f` | 로그 출력을 지속적으로 스트리밍합니다. |
| `--tail` | 마지막으로 출력된 N개의 로그만 표시합니다. |
| `--since` | 특정 시간 이후의 로그만 출력합니다. |
| `-t` | 로그 출력 시 타임스탬프를 함께 표시합니다. |

### 컨테이너 조회 (`docker ps`)
| 옵션 | 설명 |
| :--- | :--- |
| `-a` | 정지(Stopped) 및 생성(Created) 상태를 포함한 전체 목록을 출력합니다. |
| `-q` | 컨테이너 ID만 간략하게 출력합니다. |
| `--filter` | 특정 조건(예: `status=running`)에 맞는 컨테이너만 필터링합니다. |
| `--format` | Go 템플릿 문법으로 원하는 컬럼만 지정해 출력합니다. |

### `docker exec`
| 옵션 | 설명 |
| :--- | :--- |
| `-it` | 대화형(Interactive) 터미널 환경을 할당하여 내부에 접속합니다. |

## Step 5. 라이프사이클 제어 (PDF 11~12쪽)

구동 중인 컨테이너의 상태를 변경합니다.

```bash
# 1. 프로세스 다시 시작, 일시정지 및 재개
docker start web
docker pause web
docker unpause web
```

![figure8](./images/figure8.png)
---
```bash
# 2. 컨테이너 재시작
docker restart web

# 3. 30초 대기 후 정상 종료(Graceful) 및 재시작
docker restart -t 30 web
```
![figure9](./images/figure9.png)

### 유지보수 및 제어 (`docker restart`, `docker exec`)

#### `docker restart`
| 옵션 | 설명 |
| :--- | :--- |
| `-t` | 종료 대기 시간(초)을 설정합니다 (기본 10초). |

## Step 6. 데이터 백업 및 마이그레이션 (PDF 24~25쪽)

현재 상태를 백업하거나 다른 환경으로 이관하기 위한 파일 추출을 진행합니다.

```bash

# [방법 A] 컨테이너 변경사항을 새로운 이미지로 저장 (Commit)
docker commit -m "backup" web web:backup
docker images | grep web # 저장된 커밋 이미지 확인
```
![figure10](./images/figure10.png)
---
```bash

# [방법 B] 컨테이너 파일시스템 전체를 tar 파일로 추출 및 복원 (Export/Import)
docker export web > web.tar
ls -lh web.tar # 추출한 아카이브 파일 용량 확인
cat web.tar | docker import - web:imported
```
![figure11](./images/figure11.png)
---
```bash

# [방법 C] 기존 이미지를 tar 파일로 백업 및 로드 (Save/Load)
docker save -o nginx_backup.tar nginx:1.21
ls -lh nginx_backup.tar # 백업 파일 크기 확인
docker load -i nginx_backup.tar
```
![figure12](./images/figure12.png)
---
### 백업, 복원 및 삭제

#### `docker save` / `docker load`
| 옵션 | 설명 |
| :--- | :--- |
| `-o` | (`save`) 이미지를 tar 아카이브 파일로 추출합니다. |
| `-i` | (`load`) tar 파일로부터 이미지를 도커 엔진에 로드합니다. |

## Step 7. 리소스 정리 및 일괄 삭제 (PDF 13~14, 22쪽)

실습이 완료된 후 불필요한 자원을 디스크에서 완전히 정리합니다.

```bash

# 1. 단일 컨테이너 종료 (정상 종료 및 강제 종료)
docker stop web
docker ps -a
docker start web
docker kill web
```
![figure13](./images/figure13.png)
---
```bash

# 2. 실행 중인 모든 컨테이너 일괄 중지
docker stop $(docker ps -a -q)
docker ps -a
```
![figure14](./images/figure14.png)
---
```bash
# 3. 컨테이너 개별 삭제 및 강제 삭제
docker rm web
docker rm -f web-ready # Step 3에서 create로 만든 컨테이너 삭제
docker ps -a
```
![figure15](./images/figure15.png)
---
```bash

# 4. 중지된 모든 컨테이너 일괄 삭제
docker container prune -f
docker ps -a
```
![figure16](./images/figure16.png)
---
```bash
# 5. 사용하지 않는 이미지 삭제
# (Step 2에서 본인 아이디로 변경했다면 myrepo 부분을 수정해주세요)
docker rmi myrepo/nginx:v1
docker images
```
![figure17](./images/figure17.png)
---
```bash

# 6. 이름 없는(Dangling) 이미지 일괄 정리
docker image prune -f
```
![figure18](./images/figure18.png)
---

#### `docker rm/rmi` / `docker prune`
| 옵션 | 설명 |
| :--- | :--- |
| `-f` | 실행 중인 컨테이너나 사용 중인 이미지를 강제로 삭제합니다. |
| `-a` | (`image prune`) 사용되지 않는 모든 이미지를 일괄 삭제합니다. |


## 🏆 [심화 과제] 오픈소스SW 3-Tier 동아리 블로그 사수 작전!

**📝 시나리오 배경**
우리 팀은 기말 프로젝트로 '교내 중앙동아리 연합 홍보 블로그'를 워드프레스로 구축하기로 했습니다. 트래픽 폭주에 대비해 서버를 통째로 띄우지 않고, 프론트(Nginx), 백엔드(PHP), 데이터베이스(MySQL)를 각각 분리하는 실무형 '3-Tier 아키텍처'를 적용하기로 했죠!

가장 중요한 건 두 가지입니다. 
첫째, 서버가 터지더라도 동아리들이 힘들게 올린 홍보 사진과 게시글은 절대 날아가면 안 됩니다. 
둘째, 이번 과제의 핵심은 자동 네트워크 마법을 쓰지 않고, **우리가 직접 `inspect` 명령어로 흩어진 컨테이너들의 IP를 캐내어 수동으로 연결망을 완성하는 것**입니다!

자, 배운 명령어들을 총동원해서 완벽한 분산 인프라를 조립해 봅시다!


---

### 🎯 미션 요구 사항 (Tasks)

**🔥 Task 1. 데이터베이스(MySQL) 띄우고 숨겨진 IP 캐내기**
1. 호스트(Ubuntu)에 DB 데이터를 안전하게 보관할 `/opt/team-db` 폴더를 만드세요.
2. `mysql:8.0` 이미지를 사용해 `wp-db` 컨테이너를 띄웁니다.
3. 호스트를 통해 다른 컨테이너가 접속할 수 있도록 **포트(`-p 3306:3306`)**를 외부에 열어줍니다.
4. 환경변수(`-e`)로 루트 비밀번호, DB 이름(`wordpress`), 유저, 패스워드를 자유롭게 설정하세요.
5. 호스트의 폴더를 컨테이너의 `/var/lib/mysql`에 마운트(`-v`)해서 데이터를 영구 보존합니다.

**👨‍💻 Task 2. 백엔드(PHP) 띄우고 DB랑 연결하기**
1. 동아리 웹 소스와 업로드된 사진들이 저장될 `/opt/team-web` 폴더를 호스트에 만드세요.
2. `wordpress:fpm` 이미지를 사용해 `wp-php` 컨테이너를 띄웁니다.
3. Nginx가 PHP에게 전달할 수 있도록 **포트(`-p 9000:9000`)**를 열어줍니다.
4. 환경변수(`WORDPRESS_DB_HOST`)에 **`호스트_실제_IP:3306`**을 적어넣어, 백엔드가 호스트의 통로를 거쳐 DB를 찾아가게 해주세요.
5. 호스트의 웹 폴더를 컨테이너의 `/var/www/html`에 마운트합니다.

**🚀 Task 3. 프론트엔드(Nginx) 띄우고 길 안내 쪽지 쥐여주기**
1. 호스트에 `/opt/nginx-conf` 폴더를 만들고, 아래의 **[길 안내 쪽지]**를 참고하여 `default.conf` 파일을 만드세요. (쪽지 내용 중 `fastcgi_pass` 부분에 **`호스트_실제_IP:9000`**을 꼭 적어줘야 합니다!)
2. `nginx:1.21` 이미지를 사용해 `wp-web` 컨테이너를 띄웁니다. 외부 손님 접속을 위해 포트는 **`8888:80`**으로 열어주세요.
3. PHP 컨테이너와 동일한 파일을 읽을 수 있도록, `/opt/team-web` 폴더를 컨테이너의 `/var/www/html`에 마운트하세요.
4. 방금 작성한 길 안내 쪽지를 컨테이너 내부의 `/etc/nginx/conf.d/default.conf`에 마운트하여 기본 설정을 덮어씌웁니다.
   
**💣 Task 4. 대망의 오픈 테스트 & 팀플 붕괴 시뮬레이션!**
1. 브라우저에서 `localhost:8888`에 접속해 워드프레스 설치 화면이 정상적으로 뜨는지 확인하세요.
2. 동아리 홍보 글을 하나 쓰고 이미지를 업로드해 봅니다.
3. 앗, 팀원 중 한 명이 실수로 컨테이너 3개(`wp-web`, `wp-php`, `wp-db`)를 몽땅 강제 삭제(`rm -f`)해 버렸습니다!
4. 호스트 터미널에서 `/opt/team-db`와 `/opt/team-web` 폴더를 열어보세요. 컨테이너는 날아갔지만 우리의 땀과 눈물이 담긴 데이터는 무사히 살아남았음을 확인하며 안도의 한숨을 쉬어봅시다!

**✨ Task 5. 블로그 부활! (데이터 영속성 증명)**
1. 놀라지 마세요! 데이터는 호스트에 안전하게 마운트되어 있습니다.
2. IP 주소가 바뀌지 않도록 **반드시 DB ➔ PHP ➔ Nginx 순서대로**, Task 1~3에서 사용했던 똑같은 `docker run` 명령어 3줄을 그대로 다시 실행해서 컨테이너들을 살려내세요.
3. 브라우저에서 다시 `localhost:8888`을 새로고침해 보세요. 아까 작성했던 글과 사진이 마법처럼 하나도 빠짐없이 복구되어 웹사이트가 돌아가는 것을 확인해 보세요! (이것이 도커 볼륨 마운트의 진짜 위력입니다!)
---

### 💌 [참조] 프론트엔드(Nginx)를 위한 길 안내 쪽지 (`default.conf`)
작업 폴더에 파일을 만들고 아래 내용을 복사해서 붙여넣으세요.

```nginx
server {
    listen 80;
    server_name localhost;
    root /var/www/html;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        # 🚨 주의: 아래 주소에 반드시 Ubuntu 호스트의 실제 IP 주소를 적어주세요! (예: 10.0.10.55:9000)
        # 컨테이너 내부에서 localhost(127.0.0.1)를 적으면 자기 자신을 가리키게 되므로 절대 안 됩니다.
        fastcgi_pass <호스트_실제_IP>:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
