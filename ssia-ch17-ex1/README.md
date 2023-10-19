# 17장 전역 메서드 보안: 사전 및 사후 필터링

- 17.1 메서드 권한 부여를 위한 사전 필터링 적용
- 17.2 메서드 권한 부여를 위한 사후 필터링 적용
- 17.3 스프링 데이터 리포지토리에 필터링 이용
- - -
- 필터링: 메서드 호출은 허용하면서 메서드 매개 변수나 메서드의 반환값을 검증하는 방식.
  - 사전 필터링(prefiltering): 프레임워크가 메서드를 호출하기 전에 매개 변수의 값을 필터링.
  - 사후 필터링(postfiltering): 프레임워크가 메서드를 호출한 후 반환된 값을 필터링.

- 필터링은 호출 권한 부여와는 다르게, 권한 부여 규칙을 준수하지 않아도 메서드를 호출하며 예외를 던지지 않는다.

- - -
### 메서드 권한 부여를 위한 사전 필터링 적용
- @PreFilter 어노테이션
- SpEL을 이용
- 인증을 수행한 후 SecurityContext에서 이용 가능한 authentication 객체를 참조
- filterObject는 목록의 객체를 매개변수로 참조
- 필터링은 컬렉션과 배열에만 적용할 수 있다.
- 애스펙트가 컬렉션을 변경하기 때문에, 변경 불가능한 컬렉션인지 확인해야 한다. (ex. List.of())

``` java
@Service
public class ProductService {

    @PreFilter("filterObject.owner == authentication.name")
    public List<Product> sellProducts(List<Product> products) {
        return products;
    }
}
```

### 메서드 권한 부여를 위한 사후 필터링 적용
- @PostFilter 어노테이션
``` java
@Service
public class ProductService {

    @PostFilter("filterObject.owner == authentication.principal.username")
    public List<Product> findProducts() {
        List<Product> products = new ArrayList<>();

        products.add(new Product("beer", "nikolai"));
        products.add(new Product("candy", "nikolai"));
        products.add(new Product("chocolate", "julien"));

        return products;
    }
}
```

### 스프링 데이터 리포지토리에 필터링 이용
리포지토리 메서드에 @PostFilter를 적용하는 것은 기술적으로 문제가 없지만 성능 관점에서 좋지 않은 선택이다.
처음부터 필요한 것만 DB에서 검색하는 것이 레코드를 필터링하는 것보다 성능 면에서 유리하다.

- 스프링 컨텍스트에 SecurityEvaluationContextExtension 추가
``` java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class ProjectConfig {

    @Bean
    public SecurityEvaluationContextExtension securityEvaluationContextExtension() {
        return new SecurityEvaluationContextExtension();
    }
}
```

- 리포지토리 클래스의 쿼리를 SpEL 식을 이용하여 수정
``` java
public interface ProductRepository extends JpaRepository<Product, Integer> {

	@Query("SELECT p FROM Product p WHERE p.name LIKE %:text% AND p.owner=?#{authentication.principal.username}")
	List<Product> findProductByNameContains(String text);
}
```
