# 네트워크

네트워크 기초, HTTP/HTTPS, TCP/UDP, DNS와 로드밸런싱, 애플리케이션 프로토콜을 다루는 노트 모음.

## 추천 읽기 순서

1. 계층과 연결: [[OSI 7계층과 TCP-IP 4계층]] → [[3-way handshake와 4-way handshake]] → [[TCP 신뢰성 보장 메커니즘]] → [[흐름 제어와 혼잡 제어]]
2. 웹 통신: [[HTTP 메서드와 상태 코드]] → [[TLS Handshake 과정]] → [[HTTP 1.1과 HTTP 2 HTTP 3 비교]] → [[HTTP Keep-Alive와 커넥션 풀링]]
3. 요청 경로: [[DNS 조회 과정]] → [[L4와 L7 로드밸런싱]] → [[리버스 프록시]] → [[CDN 동작 원리]] → [[NAT와 방화벽 기초]]
4. 통신 방식과 인증: [[REST와 GraphQL과 gRPC 비교]] → [[gRPC]] → [[WebSocket]], 이어서 [[PKI와 인증서 체계]] → [[mTLS와 상호 인증]] → [[VPN 기초]]

## 네트워크 기초

OSI 7계층과 TCP-IP 4계층, 3-way/4-way handshake를 다룬다.

- [[OSI 7계층과 TCP-IP 4계층]]
- [[3-way handshake와 4-way handshake]]

## HTTP와 HTTPS

HTTP 메서드와 상태 코드, HTTP 1.1/2/3 비교, TLS Handshake 과정을 다룬다.

- [[HTTP 메서드와 상태 코드]]
- [[HTTP 1.1과 HTTP 2 HTTP 3 비교]]
- [[TLS Handshake 과정]]

## TCP와 UDP

TCP 신뢰성 보장 메커니즘, 흐름 제어와 혼잡 제어를 다룬다.

- [[TCP 신뢰성 보장 메커니즘]]
- [[흐름 제어와 혼잡 제어]]

## DNS와 로드밸런싱

DNS 조회 과정, L4/L7 로드밸런싱, 리버스 프록시를 다룬다.

- [[DNS 조회 과정]]
- [[L4와 L7 로드밸런싱]]
- [[리버스 프록시]]

## 애플리케이션 프로토콜

WebSocket, gRPC와 REST/GraphQL/gRPC 비교를 다룬다.

- [[WebSocket]]
- [[gRPC]]
- [[REST와 GraphQL과 gRPC 비교]]

## 네트워크 운영

CDN, 연결 재사용, NAT와 방화벽을 다룬다.

- [[CDN 동작 원리]]
- [[HTTP Keep-Alive와 커넥션 풀링]]
- [[NAT와 방화벽 기초]]

## 네트워크 보안

PKI와 인증서 체계, mTLS, VPN 기초를 다룬다.

- [[PKI와 인증서 체계]]
- [[mTLS와 상호 인증]]
- [[VPN 기초]]

## 다른 분야와 연결

- OS의 통신 API: [[소켓 프로그래밍 기초]], [[I O 멀티플렉싱 select poll epoll]]
- 프록시와 클라우드 구현: [[Nginx 리버스 프록시 설정]], [[VPC 구조]], [[서비스 메시]]
- Spring HTTP 클라이언트와 인증: [[WebClient와 RestTemplate]], [[Security Filter Chain]], [[JWT 인증]]
- API 설계와 암호화: [[RESTful API 설계 원칙]], [[대칭키와 비대칭키 암호화]]
