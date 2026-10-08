# 인프라와 운영

컨테이너, 오케스트레이션, CI/CD, 클라우드, 모니터링/로깅을 다루는 노트 모음.

## 추천 읽기 순서

1. 실행 환경: [[Docker 이미지와 레이어 구조]] → [[컨테이너 네트워킹]] → [[Kubernetes 핵심 오브젝트]] → [[Kubernetes 프로브와 안전한 종료]] → [[리소스 Requests와 Limits]] → [[오토스케일링]]
2. 클라우드와 설정: [[AWS 핵심 컴퓨트와 네트워크 서비스]] → [[VPC 구조]] → [[Terraform과 IaC 원칙]] → [[Kubernetes ConfigMap과 Secret]] → [[시크릿 관리]]
3. 배포와 보안: [[CI-CD 파이프라인 구성]] → [[배포 전략 Blue-Green Canary Rolling]] → [[클라우드 IAM 최소 권한 원칙]] → [[컨테이너 이미지 보안 스캔]]
4. 트래픽과 관측: [[Nginx 리버스 프록시 설정]] → [[서비스 메시]] → [[관측 가능성 3요소]] → [[ELK와 Prometheus Grafana]] → [[분산 트레이스 ID 전파]] → [[SLI SLO와 에러 버짓]] → [[카오스 엔지니어링과 부하 테스트]]

## 컨테이너

Docker 이미지와 레이어 구조, 컨테이너 네트워킹을 다룬다.

- [[Docker 이미지와 레이어 구조]]
- [[컨테이너 네트워킹]]

## 오케스트레이션

Kubernetes 핵심 오브젝트, 프로브와 종료, 설정 주입, 리소스 할당과 오토스케일링을 다룬다.

- [[Kubernetes 핵심 오브젝트]]
- [[Kubernetes 프로브와 안전한 종료]]
- [[Kubernetes ConfigMap과 Secret]]
- [[리소스 Requests와 Limits]]
- [[오토스케일링]]

## CI/CD

CI-CD 파이프라인 구성, 배포 전략(Blue-Green/Canary/Rolling)을 다룬다.

- [[CI-CD 파이프라인 구성]]
- [[배포 전략 Blue-Green Canary Rolling]]

## 클라우드

AWS 핵심 컴퓨트/네트워크 서비스, VPC 구조를 다룬다.

- [[AWS 핵심 컴퓨트와 네트워크 서비스]]
- [[VPC 구조]]

## 모니터링과 로깅

관측 가능성 3요소, ELK와 Prometheus/Grafana를 다룬다.

- [[관측 가능성 3요소]]
- [[ELK와 Prometheus Grafana]]

## IaC와 배포

Terraform과 시크릿 관리를 다룬다.

- [[Terraform과 IaC 원칙]]
- [[시크릿 관리]]

## 운영 심화

Nginx 리버스 프록시 설정, 서비스 메시, 카오스 엔지니어링과 부하 테스트를 다룬다.

- [[Nginx 리버스 프록시 설정]]
- [[서비스 메시]]
- [[카오스 엔지니어링과 부하 테스트]]

## 보안 운영

컨테이너 이미지 보안 스캔, 클라우드 IAM 최소 권한 원칙을 다룬다.

- [[컨테이너 이미지 보안 스캔]]
- [[클라우드 IAM 최소 권한 원칙]]

## 관측성 심화

분산 트레이스 ID 전파, SLI/SLO와 에러 버짓을 다룬다.

- [[분산 트레이스 ID 전파]]
- [[SLI SLO와 에러 버짓]]

## 다른 분야와 연결

- 네트워크 원리: [[리버스 프록시]], [[L4와 L7 로드밸런싱]], [[HTTP Keep-Alive와 커넥션 풀링]], [[mTLS와 상호 인증]]
- 관측 개념과 Spring 연동: [[분산 트레이싱]] → [[분산 트레이스 ID 전파]] → [[Micrometer와 분산 트레이싱 연동]]
- 애플리케이션 운영: [[Actuator와 헬스체크]], [[GC 튜닝]], [[커넥션 풀과 HikariCP]]
- 보안 원리와 관리: [[의존성 취약점 관리]], [[감사 로그와 보안 모니터링]], [[RBAC와 ABAC 권한 모델]]
