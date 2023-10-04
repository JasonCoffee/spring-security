# 16장 전역 메서드 보안: 사전 및 사후 권한 부여

- - -
- 16.1 전역 메서드 보안 활성화
- __16.1.1 호출 권한 부여의 이해
- __16.1.2 프로젝트에서 전역 메서드 보안 활성화
- 16.2 권한과 역할에 사전 권한 부여 적용
- 16.3 사후 권한 부여 적용
- 16.4 메서드의 사용 권한 구현
- - -

### 전역 메서드 보안
- 엔드포인트 수준에서 권한 부여 규칙을 적용하는 것이 아닌, 메서드 수준에서 권한 부여 규칙을 적용할 수 있다.
<br>→ 애플리케이션의 모든 계층에 권한 부여 규칙을 적용할 수 있다. 세부적으로, 여러 방식으로 규칙 지정이 가능하다.
- 기본적으로 비활성화. ProjectConfig에 `@EnableGlobalMethodSecurity(prePostEnabled = true)` 어노테이션 추가.

1. 메서드 호출 권한 부여 (사전 권한 부여 & 사후 권한 부여) → SpEL 기반
   - 사전 권한 부여 (Preauthorization): 메서드 호출 전에 권한 부여 규칙을 검사하는 프레임워크 `@PreAuthorize`
   - 사후 권한 부여 (Postauthorization): 메서드 호출 후에 권한 부여 규칙을 검사하는 프레임워크 `@PostAuthorize`
     <br>→ 메서드 실행은 허용하되 반환되는 내용을 검증하고 기준이 충족되지 않으면 호출자가 반환 값에 접근하지 못하게 함.

``` java
@Service
public class NameService {
	@PreAuthorize("hasAuthority('write')")
	public String getName() {
		return "Fantastico";
	}
}
```
- hasAuthority(), hasAnyAuthority(), hasRole(), hasAnyRole()
<br>→ 7장 참고

``` java
@Service
public class NameService {
	@PreAuthorize("#name == authentication.principal.username")
	public List<String> getSecretNames(String name) {
		return secretNames.get(name);
	}
}
```

``` java
@Service
public class BookService {
	@PostAuthorize("returnObject.roles.contains('reader')")
	public Employee getBookDetails(String name) {
		return records.get(name);
	}
}
```

2. 필터링 (사전 필터링 & 사후 필터링)   → 17장

### 메서드의 사용 권한 구현
- SpEL 식은 코드를 읽기 어렵게 만들고, 앱의 유지 보수 효율을 떨어뜨리므로 권장되지 않는다.
- 논리를 별도의 클래스로 만들어야 한다.

PermissionEvaluator 구현 → hasPermission()
- 객체, 사용 권한
``` java
@Service
public class DocumentService {

    @Autowired
    private DocumentRepository documentRepository;

    @PostAuthorize("hasPermission(returnObject, 'ROLE_admin')")
    public Document getDocument(String code) {
        return documentRepository.findDocument(code);
    }
}
```

``` java
@Component
public class DocumentsPermissionEvaluator implements PermissionEvaluator {

	@Override
	public boolean hasPermission(
		Authentication authentication,
		Object target,
		Object permission
	) {
		Document document = (Document)target;
		String p = (String)permission;

		boolean admin = authentication.getAuthorities()
			.stream()
			.anyMatch(a -> a.getAuthority().equals(p));

		return admin || document.getOwner().equals(authentication.getName());
	}
}
```

#### MethodSecurityExpressionHandler 정의
Spring Security가 PermissionEvaluator 구현을 인식할 수 있도록, MethodSecurityExpressionHandler 정의
``` java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class ProjectConfig extends GlobalMethodSecurityConfiguration {

	@Autowired
	private DocumentsPermissionEvaluator evaluator;

	@Override
	protected MethodSecurityExpressionHandler createExpressionHandler() {
		var expressionHandler = new DefaultMethodSecurityExpressionHandler();
		expressionHandler.setPermissionEvaluator(evaluator);
		return expressionHandler;
	}
}
```


- 객체 ID, 객체 형식, 사용 권한
``` java
@Service
public class DocumentService {

	@Autowired
	private DocumentRepository documentRepository;

	@PreAuthorize("hasPermission(#code, 'document', 'ROLE_admin')")
	public Document getDocument(String code) {
		return documentRepository.findDocument(code);
	}
}
```

``` java
@Component
public class DocumentsPermissionEvaluator implements PermissionEvaluator {

	@Autowired
	private DocumentRepository documentRepository;

	@Override
	public boolean hasPermission(
		Authentication authentication,
		Serializable targetId,
		String targetType,
		Object permission
	) {
		String code = targetId.toString();
		Document document = documentRepository.findDocument(code);

		String p = (String)permission;

		boolean admin = authentication.getAuthorities()
			.stream()
			.anyMatch(a -> a.getAuthority().equals(p));

		return admin || document.getOwner().equals(authentication.getName());
	}
}
```

### @EnableGlobalMethodSecurity
- prePostEnabled → @PreAuthorize 및 @PostAuthorize 어노테이션 활성화
- jsr250Enabled → @RolesAllowd 어노테이션 활성화
- securedEnabled → @Secured 어노테이션 활성화
