# 9장 필터 구현

HTTP 필터는 HTTP 요청을 수신하고 논리를 실행하며 체인의 다음 필터에 요청을 위임한다.


## 9.1 스프링 시큐리티 아키텍처의 필터 구현
- javax.servlet 패키지의 Filter 인터페이스를 구현.
- doFitler() 메서드를 재정의.
- 파라미터 → (ServletRequest request, ServletResponse response, FilterChain filterChain)
- 필터에는 순서 번호가 있다.
- 필터 체인에서 같은 순서에 필터를 두 개 이상 추가할 수 있다.
- 순서값이 같은 필터가 호출되는 순서는 정해지지 않는다. 보장되지 않는다.
- 필터가 호출되는 순서를 정하는 것이 유지 관리하기 쉽다.

**필터 체인**: 필터가 작동하는 순서가 정의된 필터의 모음

스프링 시큐리티의 필터 구현 예)
- BasicAuthenticationFilter
- CsrfFilter
- CorsFilter

 
## 9.2 체인에서 기존 필터 앞에 필터 추가
**RequestValidationFilter**: 헤더가 있는지 검증하는 필터

1. 필터 구현
``` java
public class RequestValidationFilter implements Filter {

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
		throws IOException, ServletException {

		var httpRequest = (HttpServletRequest)request;
		var httpResponse = (HttpServletResponse)response;

		String requestId = httpRequest.getHeader("Request-Id");
		if (requestId == null || requestId.isBlank()) {
			httpResponse.setStatus(HttpServletResponse.SC_BAD_REQUEST);
			return;
		}

		filterChain.doFilter(request, response);
	}
}
```

2. 필터 체인에 필터 추가
``` java
@Configuration
public class ProjectConfig extends WebSecurityConfigurerAdapter {

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.addFilterBefore(new RequestValidationFilter(), BasicAuthenticationFilter.class)
			.addFilterAfter(new AuthenticationLoggingFilter(), BasicAuthenticationFilter.class)
			.authorizeRequests()
			.anyRequest()
			.permitAll();
	}
}
```

## 9.3 체인에서 기존 필터 뒤에 필터 추가
**AuthenticationLoggingFilter**: 성공한 인증 이벤트를 기록하는 필터
``` java
public class AuthenticationLoggingFilter implements Filter {

	private final Logger logger = Logger.getLogger(AuthenticationLoggingFilter.class.getName());

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
		throws IOException, ServletException {

		var httpRequest = (HttpServletRequest)request;
		String requestId = httpRequest.getHeader("Request-Id");
		logger.info("Successfully authenticated request with id " + requestId);

		filterChain.doFilter(request, response);
	}
}
```

## 9.4 필터 체인의 다른 필터 위치에 필터 추가

HTTP Basic 인증 외의 다른 시나리오 예)
- 인증을 위한 정적 헤더 값에 기반을 둔 식별
- 대칭 키를 이용해 인증 요청 서명
- 인증 프로세스에 OTP(일회용 암호) 이용

**StaticKeyAuthenticationFilter**: 속성 파일에서 정적 키 값을 읽고 Authorization 헤더 값과 같은지 확인하는 필터
``` java
@Component
public class StaticKeyAuthenticationFilter implements Filter {

	@Value("${authorization.key}")
	private String authorizationKey;

	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
		throws IOException, ServletException {
		
		var httpRequest = (HttpServletRequest)request;
		var httpResponse = (HttpServletResponse)response;

		String authentication = httpRequest.getHeader("Authorization");

		if (authorizationKey.equals(authentication)) {
			filterChain.doFilter(request, response);
		} else {
			httpResponse.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
		}
	}
}
```

``` java
@Configuration
public class ProjectConfig extends WebSecurityConfigurerAdapter {

	@Autowired
	private StaticKeyAuthenticationFilter filter;

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.addFilterAt(filter, BasicAuthenticationFilter.class)
			.authorizeRequests()
			.anyRequest()
			.permitAll();
	}
}
```

## 9.5 스프링 시큐리티가 제공하는 필터 구현
스프링 시큐리티에는 Filter 인터페이스를 구현하는 여러 추상 클래스가 있다.
- GenericFilterBean: web.xml 디스크립터 파일에서 정의할 초기화 매개변수를 사용
- OncePerRequestFilter: doFilter() 메서드를 요청당 한 번만 실행하도록 보장

**AuthenticationLoggingFilter** 개선 버전 - OncePerRequestFilter 클래스 확장
: doFilterInternal() 메서드 재정의
``` java
public class AuthenticationLoggingFilter extends OncePerRequestFilter {

	private final Logger logger = Logger.getLogger(AuthenticationLoggingFilter.class.getName());

	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
		throws IOException, ServletException {

		String requestId = request.getHeader("Request-Id");

		logger.info("Successfully authenticated request with id " + requestId);

		filterChain.doFilter(request, response);
	}
}
```

**OncePerRequestFilter**
- HTTP 요청만 지원. HttpServletRequest/HttpServletResponse 로 요청을 받는다.
Filter는 형 변환해야 한다.

- 필터 적용 여부를 결정하는 논리를 구현할 수 있다.
shouldNotFilter 메서드 재정의.

- 비동기 요청이나 오류 발송 요청에는 적용되지 않는다.
shouldNotFilterAsyncDispatch() 및 shouldNotFilterErrorDispatch를 재정의.
