# 10장 CSRF 보호와 CORS 적용

## CSRF: Cross-site request forgery
- 사이트 간 요청 위조
- 애플리케이션에 로그인한 사용자로 가장해 작업을 실행할 수 있는 위조 스크립트가 포함된 페이지에 접근하도록 하는 공격 유형.

- CSRF 보호는 스프링 시큐리티에서 기본적으로 활성화된다.
- CSRF 보호는 GET, HEAD, TRACE, OPTIONS 외의 HTTP 방식은 허용하지 않음.
- 클라이언트가 이외의 방식으로 요청할 때 CSRF 토큰을 요구.
- CsrfFiter가 CSRF 보호 역할 수행.

## CSRF Filter
- 요청을 가로채고 GET, HEAD, TRACE, OPTIONS를 포함하는 HTTP 방식의 요청을 모두 허용한다.
- 이외의 다른 모든 요청에는 토큰이 포함된 헤더가 있는지 확인한다. (GET을 통해 CSRF 토큰을 생성 받았다는 것이 전제)
- 토큰은 하나의 문자열 값. X-CSRF-TOKEN=46e2b00...

- CsrfFilter는 CsrfTokenRepository 구성 요소를 이용해 토큰 생성, 토큰 저장, 토큰 검증에 필요한 CSRF 토큰 값을 관리한다.
- CsrfTokenRepository 기본 구현은 토큰을 HTTP 세션에 저장하고 랜덤 UUID로 토큰을 생성한다. (HttpSessionCsrfTokenRepository)
- CsrfFilter는 생성된 CSRF 토큰을 HTTP 요청의 _csrf 특성에 추가한다.

```
curl -v http://localhost:8080/hello

curl -X POST http://localhost:8080/hello
 -H 'Cookie: JSESSIONID=2AC6ED8126CE8306FDABAF2BE7025819'
 -H 'X-CSRF-TOKEN: 6b3ec3ce-53f4-4715-800e-bb48fb3b1f0a'
```

## Customizer 객체로 맞춤 구성
- CSRF 보호 메커니즘에서 제외할 경로 지정: ignoringAntMatchers(), ignoringRequestMatchers()

- HTTP 세션이 아닌 데이터베이스에 토큰을 관리 → 아래 구현
1. CsrfToken (DefaultCsrfToken)
2. CsrfTokenRepository (CustomCsrfTokenRepository)
﹣generateToken
﹣saveToken
﹣loadToken

![image](https://github.com/JasonCoffee/spring-security/assets/140817725/bfe2e421-d0bb-4b1e-aa11-8b44a8650f39)

``` java
public class CustomCsrfTokenRepository implements CsrfTokenRepository {

	@Autowired
	private JpaTokenRepository jpaTokenRepository;

	@Override
	public CsrfToken generateToken(HttpServletRequest httpServletRequest) {
		String uuid = UUID.randomUUID().toString();
		return new DefaultCsrfToken("X-CSRF-TOKEN", "_csrf", uuid);
	}

	@Override
	public void saveToken(CsrfToken csrfToken, HttpServletRequest httpServletRequest,
		HttpServletResponse httpServletResponse) {
		String identifier = httpServletRequest.getHeader("X-IDENTIFIER");
		Optional<Token> existingToken = jpaTokenRepository.findTokenByIdentifier(identifier);

		if (existingToken.isPresent()) {
			Token token = existingToken.get();
			token.setToken(csrfToken.getToken());
		} else {
			Token token = new Token();
			token.setToken(csrfToken.getToken());
			token.setIdentifier(identifier);
			jpaTokenRepository.save(token);
		}
	}

	@Override
	public CsrfToken loadToken(HttpServletRequest httpServletRequest) {
		String identifier = httpServletRequest.getHeader("X-IDENTIFIER");
		Optional<Token> existingToken = jpaTokenRepository.findTokenByIdentifier(identifier);

		if (existingToken.isPresent()) {
			Token token = existingToken.get();
			return new DefaultCsrfToken("X-CSRF-TOKEN", "_csrf", token.getToken());
		}

		return null;
	}
}
```

---
## CORS: Cross-Origin Resource Sharing
- 교차 출처 리소스 공유
- 특정 도메인에서 호스팅되는 웹 애플리케이션이 다른 도메인의 컨텐츠에 접근하려고 할 때 발생.
- 기본적으로 브라우저가 이러한 접근을 허용하지 않는다.
- CORS 구성을 이용하면, 리소스의 일부를 브라우저에서 실행되는 웹 애플리케이션의 다른 도메인에서 호출할 수 있다.

```
Access to XMLHttpRequest at 'http://127.0.0.1:8080/test' from origin 'http://localhost:8080' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```
→ 'Access-Control-Allow-Origin' HTTP 헤더가 없어서 응답이 수락되지 않았다는 메시지.

#### CORS 메커니즘은 HTTP 헤더를 기반으로 작동
- Access-Control-Allow-Origin
- Access-Control-Allow-Methods
- Access-Control-Allow-Headers

![image](https://github.com/JasonCoffee/spring-security/assets/140817725/f4159106-4673-480f-bc29-4f3a776604ac)

## CORS 구성
1. @CrossOrigin
```java
	@PostMapping("/test")
	@ResponseBody
	@CrossOrigin("http://localhost:8080")
	public String test() {
		logger.info("Test method called");
		return "HELLO";
	}
```

2. CorsConfigurer
``` java
@Configuration
public class ProjectConfig extends WebSecurityConfigurerAdapter {

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.cors(
			c -> {
				CorsConfigurationSource source = request -> {
					CorsConfiguration config = new CorsConfiguration();
					config.setAllowedOrigins(List.of("example.com", "example.org));
					config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
					return config;
				};
				c.configurationSource(source);
			}
		);
	}
}
```
→ 최소한 허용할 출처와 메서드를 지정해야 요청이 허용됨.
