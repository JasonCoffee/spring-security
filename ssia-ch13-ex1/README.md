# 13장: OAuth 2: 권한 부여 서버 구현

### 목차
- 13.1 맞춤형 권한 부여 서버 구현 작성
- 13.2 사용자 관리 정의
- 13.3 권한 부여 서버에 클라이언트 등록
- 13.4 암호 그랜트 유형 이용
- 13.5 승인 코드 그랜트 유형 이용
- 13.6 클라이언트 자격 증명 그랜트 유형 이용
- 13.7 갱신 토큰 그랜트 유형 이용
- - -
### 권한 부여 서버
1. 사용자를 인증
2. 클라이언트에 토큰 제공

### 그랜트
OAuth 2 프레임워크에서 클라이언트 애플리케이션이 특정 리소스에 접근하기 위한 액세스 토큰을 얻기 위해 수행하는 흐름
1. 암호 그랜트 유형
2. 승인 코드 그랜트 유형
3. 클라이언트 자격 증명 그랜트 유형
4. 갱신 토큰 그랜트 유형

### 맞춤형 권한 부여 서버 구현
Keycloak이나 Okta 같은 툴을 선택할 수도 있다.

## 13.1 맞춤형 권한 부여 서버 구현 작성
### Dependency
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-oauth2</artifactId>
</dependency>
```

### 코드
``` java
@Configuration
@EnableAuthorizationServer
public class AuthServerConfig extends AuthorizationServerConfigurerAdapter {
	@Autowired
	private AuthenticationManager authenticationManager;

	@Override
	public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
		clients.inMemory()
			.withClient("client")
			.secret("secret")
			.authorizedGrantTypes("password", "refresh_token")
			.scopes("read");
	}

	@Override
	public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
		endpoints.authenticationManager(authenticationManager);
	}
}
```

- `AuthorizationServerConfigurerAdapter` 클래스 확장
- `@EnableAuthorizationServer` 어노테이션 지정: OAuth2 권한 부여 서버에 관한 구성 활성화

## 13.2 사용자 관리 정의
### 코드
``` java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
	@Bean
	public UserDetailsService uds() {
		var uds = new InMemoryUserDetailsManager();

		var u = User.withUsername("john")
			.password("12345")
			.authorities("read")
			.build();

		uds.createUser(u);

		return uds;
	}

	@Bean
	public PasswordEncoder passwordEncoder() {
		return NoOpPasswordEncoder.getInstance();
	}

	@Override
	@Bean
	public AuthenticationManager authenticationManagerBean() throws Exception {
		return super.authenticationManagerBean();
	}
}
```

- `WebSecurityConfigurerAdapter` 클래스 확장
- `AuthenticationManager` 인스턴스를 스프링 컨텍스트에 빈으로 추가
- `AuthServerConfig` 클래스에서 `AuthenticationManager` 사용

## 13.3 권한 부여 서버에 클라이언트 등록
- 12장에서는 Github을 인증 서버로 이용
- `ClientDetails` (`BaseClientDetails`): 권한 부여 서버에서 클라이언트를 정의
- `ClientDetailsService` (`InMemoryClientDetailsService`): 클라이언트 구성을 정의. `ClientDetails`를 검색.
- `AuthServerConfig` 참고

## 13.4 암호 그랜트 유형 이용 (AuthorizedGrantTypes: password)
![image](https://github.com/JasonCoffee/spring-security/assets/140817725/39248a70-8f83-4d41-a8f1-bcd5cacde13c)
- grant_type: password 값을 가진다.
- username&password: 사용자 자격 증명
- scope: 허가된 권한

### 실행 예시
```
$ curl -v -X POST -u client:secret 'http://localhost:8080/oauth/token?grant_type=password&username=john&password=12345&scope=read'
```
``` json
{
    "access_token": "03d05ebe-c697-4a6b-8853-796fd3c2691c",
    "token_type": "bearer",
    "refresh_token": "50f868cf-8dfc-49fb-8f62-1d95db229808",
    "expires_in": 43151,
    "scope": "read"
}
```

## 13.5 승인 코드 그랜트 유형 이용 (AuthorizedGrantTypes: authorization_code)
![image 2](https://github.com/JasonCoffee/spring-security/assets/140817725/b8fcf72d-0e80-4f40-b843-12894f57a6c8)
- OAuth 2 그랜트 유형 중 가장 많이 이용됨.
- 리디렉션 URI 제공: 권한 부여 서버가 인증을 마친 사용자를 리디렉션할 URI.
- 권한 부여 서버는 리디렉션 URI를 호출할 때 액세스 코드도 제공.

### 코드
``` java
@Configuration
@EnableAuthorizationServer
public class AuthServerConfig extends AuthorizationServerConfigurerAdapter {

	@Autowired
	private AuthenticationManager authenticationManager;

	@Override
	public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
		clients.inMemory()
			.withClient("client")
			.secret("secret")
			.authorizedGrantTypes("authorization_code", "refresh_token")
			.scopes("read")
			.redirectUris("http://localhost:9090/home");
	}

	@Override
	public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
		endpoints.authenticationManager(authenticationManager);
	}
}

```

- 클라이언트별로 다른 그랜트 유형을 설정할 수 있다.
- 한 클라이언트를 위한 여러 그랜트 유형을 설정할 수 있다.
- 사용자 자격 증명과 마찬가지로 클라이언트 자격 증명은 공유하면 안 된다.
- 각 클라이언트 애플리케이션 개별 등록으로 분리/격리 (각각 컨트롤 가능, 영향성 분리)

### 실행 예시
```
$ http://localhost:8080/oauth/authorize?response_type=code&client_id=client&scope=read

→ http://localhost:9090/home?code=kUn2iX

$ curl -v -X POST -u client:secret 'http://localhost:8080/oauth/token?grant_type=authorization_code&scope=read&code=kUn2iX'
```

``` json
{
    "access_token": "6c46d202-dc82-4d9f-94bc-5f8c579f7d21",
    "token_type": "bearer",
    "refresh_token": "943e5c58-0eba-49ff-bed4-cd30a67f0f7b",
    "expires_in": 43199,
    "scope": "read"
}
```

``` json
{
    "error": "invalid_grant",
    "error_description": "Invalid authorization code: abcdef"
}
```

## 13.6 클라이언트 자격 증명 그랜트 유형 이용 (AuthorizedGrantTypes: client_credentials)
![image 3](https://github.com/JasonCoffee/spring-security/assets/140817725/7b0e36c6-ea85-48f0-93eb-ef3405984c3c)
- 백엔드-대-백엔드 권한 부여에 이용.
- 말 그대로 클라이언트만 자격 증명. 사용자와 무관.
- 클라이언트가 접근해야 하는 엔드포인트를 보호하는데 이용. ex) 서버 상태를 알려주는 엔드포인트.
- 사용자 자격 증명이 필요한 범위에 접근을 허용하면 안 된다.

### 코드
``` java
@Configuration
@EnableAuthorizationServer
public class AuthServerConfig extends AuthorizationServerConfigurerAdapter {

	@Autowired
	private AuthenticationManager authenticationManager;

	@Override
	public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
		clients.inMemory()
			.withClient("client")
			.secret("secret")
			.authorizedGrantTypes("client_credentials")
			.scopes("info");
	}

	@Override
	public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
		endpoints.authenticationManager(authenticationManager);
	}
}
```

### 실행 예시
```
$ curl -v -X POST -u client:secret 'http://localhost:8080/oauth/token?grant_type=client_credentials&scope=info'
```
``` json
{
    "access_token": "f4f590cf-0d1e-45fc-948f-dc67d6356c20",
    "token_type": "bearer",
    "expires_in": 43199,
    "scope": "info"
}
```

## 13.7 갱신 토큰 그랜트 유형 이용 (AuthorizedGrantTypes: refresh_token)
![image 4](https://github.com/JasonCoffee/spring-security/assets/140817725/bb0f108f-1087-4d43-9f1e-9c812af50b1d)
- 갱신 토큰을 승인 코드/암호 그랜트 유형 등 다른 그랜트 유형과 함께 이용하면 여러 이점이 있다.
- 클라이언트 등록 부분에 `refresh_token` 추가하여 권한 부여 서버가 액세스 토큰과 갱신 토큰을 발행하도록 할 수 있다.
- 갱신 토큰을 이용하면 클라이언트가 사용자를 다시 인증할 필요 없이 새 액세스 토큰을 얻을 수 있다.

### 코드
``` java
@Configuration
@EnableAuthorizationServer
public class AuthServerConfig extends AuthorizationServerConfigurerAdapter {
	@Autowired
	private AuthenticationManager authenticationManager;

	@Override
	public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
		clients.inMemory()
			.withClient("client")
			.secret("secret")
			.authorizedGrantTypes("password", "refresh_token")
			.scopes("read");
	}

	@Override
	public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
		endpoints.authenticationManager(authenticationManager);
	}
}
```

### 실행 예시
```
$ curl -v -X POST -u client:secret 'http://localhost:8080/oauth/token?grant_type=password&username=john&password=12345&scope=read'

```
``` json
{
    "access_token": "c2910004-8d40-49c3-be20-8e6859f091e5",
    "token_type": "bearer",
    "refresh_token": "cc8be216-c7b8-4ea4-9bb8-930cab87a6f9",
    "expires_in": 43199,
    "scope": "read"
}
```
