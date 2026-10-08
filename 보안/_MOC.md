# 보안

인증과 인가, 암호화 기초, 애플리케이션 보안을 다루는 노트 모음. Spring Security의 구체적인 구현(JWT, Security Filter Chain 등)은 [[Spring/_MOC|Spring]] 도메인을 참고.

## 추천 읽기 순서

1. 인증과 인가: [[세션 기반 인증과 토큰 기반 인증 비교]] → [[RBAC와 ABAC 권한 모델]] → [[SSO와 SAML과 OIDC]]
2. 암호화와 저장: [[대칭키와 비대칭키 암호화]] → [[해시 함수와 솔팅]]
3. 애플리케이션 방어: [[OWASP Top 10]] → [[SQL Injection과 방어]] → [[XSS와 CSRF 방어]] → [[SSRF와 아웃바운드 요청 검증]]
4. 보안 운영: [[의존성 취약점 관리]] → [[감사 로그와 보안 모니터링]]

## 인증과 인가

세션 기반 인증과 토큰 기반 인증 비교, RBAC와 ABAC 권한 모델, SSO와 SAML/OIDC를 다룬다.

- [[세션 기반 인증과 토큰 기반 인증 비교]]
- [[RBAC와 ABAC 권한 모델]]
- [[SSO와 SAML과 OIDC]]

## 암호화 기초

대칭키와 비대칭키 암호화, 해시 함수와 솔팅을 다룬다.

- [[대칭키와 비대칭키 암호화]]
- [[해시 함수와 솔팅]]

## 애플리케이션 보안

OWASP Top 10, SQL Injection과 방어, XSS와 CSRF 방어, SSRF와 아웃바운드 요청 검증을 다룬다.

- [[OWASP Top 10]]
- [[SQL Injection과 방어]]
- [[XSS와 CSRF 방어]]
- [[SSRF와 아웃바운드 요청 검증]]

## 보안 운영 실전

의존성 취약점 관리, 감사 로그와 보안 모니터링을 다룬다.

- [[의존성 취약점 관리]]
- [[감사 로그와 보안 모니터링]]

## 다른 분야와 연결

- Spring 구현: [[Security Filter Chain]], [[JWT 인증]], [[OAuth2와 소셜 로그인]], [[Method Security와 CSRF 방어]] — [[Spring/_MOC|Spring 목차]]
- 전송 구간 보안: [[TLS Handshake 과정]], [[PKI와 인증서 체계]], [[mTLS와 상호 인증]]
- 운영 환경의 권한과 비밀정보: [[클라우드 IAM 최소 권한 원칙]], [[시크릿 관리]], [[Kubernetes ConfigMap과 Secret]]
- 공급망과 관측: [[컨테이너 이미지 보안 스캔]], [[CI-CD 파이프라인 구성]], [[관측 가능성 3요소]]
