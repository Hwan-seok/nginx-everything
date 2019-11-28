# Nginx

## 개요
- Event-Driven 방식으로 클라 요청 처리하는 웹 서버
- 이벤트 기반이라서 apache보다 좋은 성능 가짐

## 쓰임새
- 리버스 프록싱
	- 외부에서 내부 서버 알 수 없으므로 보안 굿
	- 로드 밸런싱
	- ** forward(순방향) proxy는 내부망에서 외부로 나갈 때 프록시 거치는것
- 정적파일 서빙
### 명령어
- `nginx -s SIGNAL`
#### 시작
- `nginx`
#### 종료
- `nginx -s stop`
- fast shutdown
#### 조심스럽게 종료
- `nginx -s quit`
- graceful shutdown 
- 현재 연결된 요청까지 처리하고 shutdown
#### 조심스럽게 리로드
- `nginx -s reload`
- graceful shutdown + 변경된 conf파일 적용 후 start
#### 로그파일 재오픈
`nginx -s reopen`
- reopen 참조 
- [what-does-nginx-s-reopen-do](https://unix.stackexchange.com/questions/186807/what-does-nginx-s-reopen-do)
- [nginx log file rotate](https://www.lesstif.com/pages/viewpage.action?pageId=75956229) 

--- 
## conf 파일 분석
### Simple directive
- 한 줄짜리 지시어 -> 꼭 세미콜론(;) 붙혀야 함
```nginx
user www-data; # 시스템의 어떤 사용자가 서버 동작시킬지 기술
# root 주면 워커 프로세스가 root 권한으로 동작하므로 위험  
# `useradd --shell /usr/sbin/nologin www-data` 이친구론 shell 접근 불가, nginx 구동용

worker_processes auto; # 워커 프로세스 생성 개수 - CPU 코어 수 맞추면 좋음
# worker_processes 4;

client_max_body_size 10M; # 업로드 최대 용량 (기본값 1MB)

log_not_found off; # 존재하지 않는 파일 요청 404를 로그 파일에 기록할 것인지
# log_not_found /logs/log/samplefile

pid /run/nginx.pid; # nginx pid 파일 위치
include /etc/nginx/modules-enabled/*.conf; # 외부 conf import

error_log /var/log/hwanseok.org.error debug;
```
### Block directive
- 블록 지시어
	- 블록 내부에 다수의 단순|블록 지시어 포함
#### events
- 네트워크 동작방법 설정값
```nginx
events {
	worker_connections 1024;
	# 하나의 워커 프로세스에서 몇개의 접속 동시 처리할 것인가
	# worker_connections * worker_processes = 4 * 1024 개의 동시 커넥션 
}
```
#### http - server - location
```nginx
http {
	server {
		server_name hwanseok.org hs.com *.hs.org;  #호스트명(주로 도메인)
		# server_name localhost;
		include module_name/*;
		location {
		
		}
	}
}
```
#### http
- server, location의 root 블록 
	- 하위 블록에게 속성 상속
	- 하위 블록에서 중복 선언된 지시어는 상위 선언 무시
#### server
- web site 하나 선언하는데 사용
	- 가상 호스팅(virtual host)
	- `http://HS.org`와 `http://sy.org`를 하나의 노드에서 운용 가능
```nginx
server { 
	listen  80;
	server_name example.org www.example.org;
	...
}
```
```nginx
server {
		listen 80;	      # 80포트에 대해서
		listen [::]:80;	 # 80포트에 대해서
		# https로 리다이렉트
		return 301 https://$host$request_uri;
	}
```
#### location
- 특정 서버의 특정 URL을 처리하는 방법을 정의
- URI 경로 일부이거나 정규식이 될 수 있음
- 가장 긴 경로부터 비교하여 배정함
```nginx
# 요청 경로에 images포함되면 /data, 파일 이름은 images 나머지는 다른 호스트
server {
	root 가서는 안될곳;  
	access_log 요런곳;  
	location /images/ { 
		access_log 요런저런곳;  
		root /data; 
		# http://hs.org/images/index.html 요청시 /data/index.html 반환
	} 
	location ~ \.(gif|jpg|png)$ {  
		access_log 저런곳;  
		# try_files $uri /index.html =404; 차례로 파일 있는지 확인
		root /data/images;
	}
	location / { 
		
		proxy_pass http://www.example.com;
	} 
}
```
```nginx
# 리다이렉션
# last, break, redirect, permanent 플래그 있음
요청 URL: http://test.com/docs/readme.html

location ~ /docs/readme.html {
	rewrite ^ http://test.com/files/docs/readme.html;
}
```
```nginx
# 내부 리다이렉션 - 브라우저는 모름
요청 URL: http://test.com/docs/readme.html
실제 파일 URL: http://test.com/files/docs/readme.html

location /docs/ {
	rewrite ^/docs/(.*)$ /files/docs/$1;
}
```
```nginx
favicon때매 화나셨죠? 이걸로 해결하세요!
location = /favicon.ico {
	return 204;
}
```
#### upstream
- 부하 분산, 속도 개선
- 한 대의 웹 서버에 여러 대의 WAS 서버
![](https://s3.ap-northeast-2.amazonaws.com/opentutorials-user-file/module/384/1391.gif)
-   ip_hash : 같은 ip 요청은 같은 서버가 처리
-   weight=n : 서버의 요청 처리 비중
-   max_fails=n : 지정한 횟수만큼 요청 실패시 죽은 것으로 간주
-   fail_timeout=n : 해당 시간만큼 timout이 max_fails 만큼 반복시 죽은 것으로 간주
-   down : 해당 서버를 사용하지 않게 지정 ip_hash 일때만 유효
-   backup : 모든 서버 다운시 활성화
```nginx
http {
	server {
		location / {
			proxy_pass backend;
			proxy_http_version 1.1;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection 'upgrade';
			proxy_set_header Host $host;
			proxy_cache_bypass $http_upgrade;
		}
	}
	upstream backend {  
		ip_hash;
		# least_conn;
		# default round-robin
		
		server 192.168.125.142:9000 weight=3; # 다른 서버에 비해 3배 자주 사용
		server 192.168.125.143:9000; 
		server 192.168.125.144:9000 max_fails=5 fail_timeout=30s; # 30초 timeout 5번 반복되면 죽은것으로 간주해 요청 보내지 않음
		server unix:/var/run/php5-fpm.sock backup; # 나머지 서버 불능시 활성화 - 평소엔 비활성화
	}
}
```
#### mail
```nginx
mail {

}
```
#### stream
- TCP,  UDP 
```nginx
stream {

}
```
### 연산자
- (!)= 
- (!)~
	- 정규 표현식 매칭
	- $1 = 첫 번째로 매칭된 값 (n 가능)
	- location ~ "^/user/(?<user_id>[0-9]+)/(.*)$ 이면 첫번째 괄호가 $1, 두번째가 $2
```nginx
#html로 끝나는($) 경로 접근에 대하여 처리
location ~ /.html$ { 
	error_log /var/log/hwanseok.org.error debug;
}
```
- (!)~* 
	- 대소문자 구별 안하는 정규 표현식 매칭
### 변수
다음 URL 접근시 환경 변수
http://hwanseok.org:80/production/module/index.html?type=module&id=12
-   $host : hwanseok.org
-   $uri : /production/module/index.html
-   $args : type=module&id=12
-   $server_addr : 115.68.24.88
-   $server_name : localhost
-   $server_port : 80
-   $server_protocol : HTTP/1.1
-   $arg_type : module
-   $request_uri : /production/module/index.php?type=module&id=12
-   $request_filename : /usr/local/nginx/html/production/module/index.html
-  $arg_type : module

$content_type
$cookie_COOKIENAME
$scheme - http/https
### 참조
- 파일 위치 - linux
```
- /usr/local/nginx/conf
- /etc/nginx
- /usr/local/etc/nginx
```

### 참고한 블로그 및 사진 출처
- opentutorials.org
- [https://gonna-be.tistory.com/20](https://gonna-be.tistory.com/20)
- [https://opennote46.tistory.com/214](https://opennote46.tistory.com/214)
- [https://github.com/CreatiCoding/KOREAWIKI/blob/master/nginx%20%ED%8A%9C%ED%86%A0%EB%A6%AC%EC%96%BC.md](https://github.com/CreatiCoding/KOREAWIKI/blob/master/nginx%20%ED%8A%9C%ED%86%A0%EB%A6%AC%EC%96%BC.md)
- [https://www.netguru.com/codestories/nginx-tutorial-basics-concepts](https://www.netguru.com/codestories/nginx-tutorial-basics-concepts)
- [https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
- [https://www.lesstif.com/pages/viewpage.action?pageId=35357063](https://www.lesstif.com/pages/viewpage.action?pageId=35357063)
