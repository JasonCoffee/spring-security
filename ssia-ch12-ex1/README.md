## OAuth 2 
- 권한 부여 프레임워크
- 타사 웹사이트가 리소스에 접근할 수 있게 허용하는 것이 주 목적.
- 네트워크를 통해 자격 증명이 자주 공유하면 보안을 약화하기 때문에
- 애플리케이션 아키텍처에서 제거하고 별도의 시스템에서 사용자 자격 증명을 관리하는 것이 바람직.
- 권한 부여 서버를 별도로.

### 주요 구성 요소
- 사용자
  - 사용자 이름/암호로 증명.
- 클라이언트
  - 사용자를 대신해 리소스에 접근하는 애플리케이션. (Client-id, client-secret)
- 리소스 서버
  - 사용자가 소유한 리소스를 제공하는 서버.
- 권한 부여 서버

### OAuth 2 그랜트 유형
#### 승인 코드 그랜트
- 사용자가 직접 권한 부여 서버에서 인증하여 클라이언트가 액세스 토큰을 얻는 방법
<img width="923" alt="image" src="https://github.com/JasonCoffee/spring-security/assets/140817725/b056694c-917c-46cd-9eb7-40ef9fa15964">

##### 클라이언트가 권한 부여 서버로 사용자를 리디렉션할 때 주는 정보 1
- response type
- client_id
- redirect_uri
- scope
- state

* 권한 부여 서버가 클라이언트에게 바로 액세스 토큰을 반환하는 암시적 그랜트 유형(implicit grant type)은 권장되지 않음.
##### 클라이언트가 권한 부여 서버에 제공하는 정보 2
- code
- client_id, client_secret
- redirect_uri
- grant_type

#### 암호 그랜트 (리소스 소유자 자격 증명 그랜트)
- 사용자가 자신의 자격 증명을 클라이언트와 공유하는 방법
- 클라이언트와 권한 부여 서버를 같은 조직에서 구축하고 관리할 때만 이용
<img width="931" alt="image" src="https://github.com/JasonCoffee/spring-security/assets/140817725/dcdf3da7-24e7-4c4f-bb44-19c723050167">

#### 클라이언트 자격 증명 그랜트
- 클라이언트가 자격 증명만으로 인증하여 토큰을 얻는 방법
- 사용자 자격 증명이 필요하지 않다.
- 사용자의 리소스가 아닌 리소스 서버의 엔드포인트를 호출할 때 사용
<img width="938" alt="image" src="https://github.com/JasonCoffee/spring-security/assets/140817725/99aa3422-ce2e-4fa3-ae86-fbcc03986962">

#### 갱신 토큰
- 가능한 토큰이 최소한의 수명을 가지도록 해야 한다.
- 액세스 토큰 + 갱신 토큰
- (사용자가 아닌) 클라이언트가 갱신 과정 수행

#### OAuth 2의 허점
- 클라이언트에서 CSRF 이용
- 클라이언트 자격 증명 도용
- 토큰 재생
- 토큰 하이재킹
  

OAuth2LoginAuthenticationFilter 필터





