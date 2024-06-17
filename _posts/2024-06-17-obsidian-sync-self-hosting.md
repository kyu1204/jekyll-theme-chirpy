---
title: couchdb를 이용해 obisidian sync 셀프 호스팅 하기
categories: [obsidian]
tags: [couchdb, obsidian, self-hosting]
date: 2023-06-17 00:00:00 +09:00
image: obsidian-sync.png
img_path: /assets/img/obsidian-sync/
---
## 개요
기존에 obsidian sync 방법으로 iCloud(~~Fxxk iCloud~~) 를 사용중이였는데, 이기종간 sync 속도가 진짜 절망적이였습니다. 
![주디의 마음](https://media4.giphy.com/media/3oz8xOu5Gw81qULRh6/giphy.gif?cid=6c09b952842lxohmshi8wodyeh80tc9ehc8a4jwh40lj3sgc&ep=v1_internal_gif_by_id&rid=giphy.gif&ct=g)
<p style="text-align:center; font-size:80%; color:gray">주디의 마음</p>

Obsidian의 가장 큰 장점중 하나가 무엇인가, 바로 다양한 커뮤니티 플러그인..! 분명 self-host 할 수 있는 obsidian sync가 있을거라고 믿어의심치 않았고, 결국 찾아냈습니다
바로 ["Self-hosted LiveSync"](obsidian://show-plugin?id=obsidian-livesync)

![드디어 찾았다](https://media0.giphy.com/media/3o6Mb3Feec33LawNdm/giphy.gif?cid=6c09b9525w64o62fwvtgdf3r603nil8fnx1vw8bj781705ck&ep=v1_internal_gif_by_id&rid=giphy.gif&ct=g)
<p style="text-align:center; font-size:80%; color:gray">드디어 찾았다</p>

## Prerequisite
셀프 호스팅으로 필요로 하는 서버 구성은 couch db가 필요합니다
다행스럽게도, 도커이미지가 있었고 필요한 구성은 도커 컴포즈로 만들어 준비했습니다.
그래서 당연하게도 셀프호스팅 서버에는 도커가 설치되어 있어야합니다.
### 필요한 준비물
- 셀프 호스팅이 가능한 서버
- 도커
- 도커 컴포즈 파일
- init.sh 파일
## Install
필자는 셀프호스팅 서버는 light sail에 띄워둔 인스턴스를 사용했습니다. 도커와 웹 서버 서빙만 가능하다면 아무거나 상관없어요.
### 1. 도커 컴포즈 파일
작성한 도커 컴포즈 파일은 다음과 같습니다
```yaml
version: '3.8'

services:
couchdb:
    image: couchdb
    container_name: couchdb-for-ols
    environment:
- COUCHDB_USER=minkyu
- COUCHDB_PASSWORD=kawApVAd&qAmS7
    volumes:
- /home/ubuntu/couchdb/couchdb-data:/opt/couchdb/data
- /home/ubuntu/couchdb/couchdb-etc:/opt/couchdb/etc/local.d
    ports:
- "5984:5984"
    restart: always
```
### 2. 환경변수 설정
```bash
export hostname="localhost:5984"
export username="사용할 username"
export password="사용할 password"
```
### 3. init.sh
init.sh 는 공식 문서를 참조했습니다.

```shell
#!/bin/bash
if [[ -z "$hostname" ]]; then
        echo "ERROR: Hostname missing"
exit 1
fi
if [[ -z "$username" ]]; then
echo "ERROR: Username missing"
exit 1
fi

if [[ -z "$password" ]]; then
echo "ERROR: Password missing"
exit 1
fi

echo "-- Configuring CouchDB by REST APIs... -->"

until (curl -X POST "${hostname}/_cluster_setup" -H "Content-Type: application/json" -d "{\"action\":\"enable_single_node\",\"username\":\"${username}\",\"password\":\"${password}\",\"bind_address\":\
"0.0.0.0\",\"port\":5984,\"singlenode\":true}" --user "${username}:${password}"); do sleep 5; done
until (curl -X PUT "${hostname}/_node/nonode@nohost/_config/chttpd/require_valid_user" -H "Content-Type: application/json" -d '"true"' --user "${username}:${password}"); do sleep 5; done
until (curl -X PUT "${hostname}/_node/nonode@nohost/_config/chttpd_auth/require_valid_user" -H "Content-Type: application/json" -d '"true"' --user "${username}:${password}"); do sleep 5; done
until (curl -X PUT "${hostname}/_node/nonode@nohost/_config/httpd/WWW-Authenticate" -H "Content-Type: application/json" -d '"Basic realm=\"couchdb\""' --user "${username}:${password}"); do sleep 5; do
ne
until (curl -X PUT "${hostname}/_node/nonode@nohost/_config/httpd/enable_cors" -H "Content-Type: application/json" -d '"true"' --user "${username}:${password}"); do sleep 5; done
until (curl -X PUT "${hostname}/_node/nonode@nohost/_config/chttpd/enable_cors" -H "Content-Type: application/json" -d '"true"' --user "${username}:${password}"); do sleep 5; done
until (curl -X PUT "${hostname}/_node/nonode@nohost/_config/chttpd/max_http_request_size" -H "Content-Type: application/json" -d '"4294967296"' --user "${username}:${password}"); do sleep 5; done
until (curl -X PUT "${hostname}/_node/nonode@nohost/_config/couchdb/max_document_size" -H "Content-Type: application/json" -d '"50000000"' --user "${username}:${password}"); do sleep 5; done
until (curl -X PUT "${hostname}/_node/nonode@nohost/_config/cors/credentials" -H "Content-Type: application/json" -d '"true"' --user "${username}:${password}"); do sleep 5; done
until (curl -X PUT "${hostname}/_node/nonode@nohost/_config/cors/origins" -H "Content-Type: application/json" -d '"app://obsidian.md,capacitor://localhost,http://localhost"' --user "${username}:${pass
word}"); do sleep 5; done

echo "<-- Configuring CouchDB by REST APIs Done!"
```

실행 권한을 준 뒤 쉘 스크립트를 실행시키고 나서 다음 결과가 뜨면 셋팅 완료 입니다.
```bash
$ chmod +x init.sh
$ ./init.sh
...
{"ok":true}
...
```

### 4. 서버 호스팅
필자는 별도 도메인이 있어서 cloudflare 에 서브 도메인으로 추가 후 nginx 에서 서빙하였다.
localhost:포트 가 뜬 상황이기 때문에 해당 주소를 프록시만할 수 있으면 어떤방법을 사용해도 무방하다.(시놀리지 NAS 에도 설치 가능할듯..!?)
이렇게 설치하면 서버 준비는 끝이다.
### 5. Obsidian Self-hosted LiveSync 설치
![1.png](1.png)
[설치 링크](obsidian://show-plugin?id=obsidian-livesync)

![2.png](2.png)
설치 후 Minimal setup 선택

![4.png](4.png)
셀프 호스팅 서버 주소, 아이디, 비밀번호, 사용할 디비 이름 입력 후 check 버튼으로 연결 확인

![7.png](7.png)
Presets은 LiveSync 선택 (다른 이벤트도 있지만 우린 실시간 싱크를 원하니까⚡️)
설정을 완료하고 나면 자동으로 pulling이 시작되는데 **매우매우매우** 오래걸리니까 차분히 유튜브 영상 하나 보고오시면 됩니다.

![6.png](6.png)
그 후 Encrypt your settings이 뜨는데 E2E Encrypt 설정을 하지 않았기 때문에 cancle 해도 됩니다.

![9.png](9.png)
기나긴 pulling이 끝나고 나면 hidden file (plugin, setting, etc...)에 대한 설정이 나오는데, fetch를 선택하면 됩니다. 

![[8.png]]
**혹시라도 hidden file에 대한 팝업이 안뜬다면 설정에서 직접 fetch를 해봅시다**

![10.png](10.png)
**LiveSync가 풀려있는 경우가 몇번 있는데 한번씩 다시 설정해주면 됩니다.**

## Test
### 싱크 테스트
기본 싱크할 오리지널 vault가 모두 셋업이 완료되고나면 새로운 클라이언트에서 새로운 vault를 만든 뒤 Self-Hosted LiveSync 만 설치하고 동일하게 셋업 하시고 기나긴 pulling 이후 뜨는 hidden file 에 대한 설정은 fetch만 하시면 됩니다.
![gif.gif](gif.gif)

## 마치며
Obsidian Sync 에 대해 셀프 호스팅을 직접 해봤는데, 결제해서 사용하면 이런 수고스러움 없이 바로 sync 사용과 웹 퍼블리싱 기능까지 사용할 수 있으니 모두 ~~그냥 돈주고 결제 합시다~~

다만 본인이 놀고 있는 서버가 있고, 개인 도메인이 있는 진성 IT 덕후라면 한번 시도해볼만합니다. iCloud에 비하면 3G -> 5G 급 sync 속도변화가 일어나고 obsidian-git 처럼 매번 커밋 풀 할 필요 없이 live sync까지 되니 한번 구축해두면 정말 잘 쓸 수 있을 것 같아 강추드립니다.

## Referance
- https://github.com/vrtmrz/obsidian-livesync
- https://gist.github.com/yeongu-dev/08dfbadc9a7c79d23b1022a30dd7eebe