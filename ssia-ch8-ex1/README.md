---
# 8장 권한 부여 구성: 제한 적용
특정한 요청 그룹에만 matcher 메서드를 사용하여 권한 부여 제약 조건을 적용하는 방법.

---
### 스프링 시큐리티의 세 유형의 Matcher 메서드
- MVC matcher
 
  : 경로에 MVC 식을 이용해 엔드포인트를 선택한다.
- 앤트 matcher

  : 경로에 앤트 식을 이용해 엔드포인트를 선택한다.
- 정규식 matcher

  :경로에 정규식을 이용해 엔드포인트를 선택한다.

---
## MVC Matcher
- `mvcMatchers(HttpMethod method, String... patterns)`

  제한을 적용할 HTTP 방식과 경로를 모두 지정 가능.
  같은 경로에 대해 HTTP 방식별로 다른 제한을 적용할 때 유용.
  
- `mvcMatchers(String... patterns)`

  경로만을 기준으로 권한 부여 제한을 적용
  자동으로 해당 경로의 모든 HTTP 방식에 제한이 적용.

---
  ``` java
    protected void configure(HttpSecurity http) throws Exception {
        http.httpBasic();

        http.authorizeRequests()
                .mvcMatchers("/hello").hasRole("ADMIN")
                .mvcMatchers("/ciao").hasRole("MANAGER")
                .anyRequest().permitAll();
    }
```
  - 나머지 모든 엔드포인트에 대해 모든 요청을 허용

    
---
``` java
    protected void configure(HttpSecurity http) throws Exception {
        http.httpBasic();

        http.authorizeRequests()
                .mvcMatchers("/hello").hasRole("ADMIN")
                .mvcMatchers("/ciao").hasRole("MANAGER")
                .anyRequest().authenticated();
    }
```
- 인증된 사용자에게만 나머지 모든 요청을 허용

---
``` java
    protected void configure(HttpSecurity http) throws Exception {
        http.httpBasic();

        http.authorizeRequests()
                .mvcMatchers(HttpMethod.GET, "/a").authenticated()
                .mvcMatchers(HttpMethod.POST, "/a").permitAll()
                .anyRequest().denyAll();
    }
```
- GET /a → 사용자 인증 필요
- POST /a → 요청 모두 허용
- 다른 경로에 대한 모든 요청 거부

---
``` java
    protected void configure(HttpSecurity http) throws Exception {
        http.httpBasic();

        http.authorizeRequests()
                .mvcMatchers( "/a/b/**").authenticated()
                .anyRequest().permitAll();
    }
```
- 경로식 사용. 스프링 MVC에서 앤트의 패턴 일치 구문을 차용.
- /a/b가 붙은 모든 경로에 대해 인증 필요
- ** 연산자: 수 제한 없는 경로, * 연산자: 경로 하나만 지정.

---
``` java
    protected void configure(HttpSecurity http) throws Exception {
        http.httpBasic();

        http.authorizeRequests()
                .mvcMatchers( "/product/{code:^[0-9]*$}").permitAll()
                .anyRequest().denyAll();
    }
```
- 길이와 관계없이 숫자를 포함하는 문자열을 허용.

---
## Ant matcher
- `antMatchers(HttpMethod method, String patterns)`

  제한을 적용할 HTTP 방식과 경로를 참조할 앤트 패턴을 모두 지정 가능.
  같은 경로 그룹에 대해 HTTP 방식별로 다른 제한을 적용할 때 유용.

- `antMatchers(String patterns)`

  경로만을 기준으로 권한 부여 제한을 적용할 때 이용 가능.
  모든 HTTP 방식에 자동으로 제한이 적용.

- `antMatchers(HttpMethod method)`
  
  = antMatchers(httpMethod, "/**")
  경로와 관계없이 특정 HTTP 방식을 지정 가능.

---
``` java
    protected void configure(HttpSecurity http) throws Exception {
        http.httpBasic();

        http.authorizeRequests()
                .mvcMatchers( "/hello").authenticated();
    }
```
- /hello/ 에도 적용

---
``` java
    protected void configure(HttpSecurity http) throws Exception {
        http.httpBasic();

        http.authorizeRequests()
                .antMatchers( "/hello").authenticated();
    }
```
- /hello/ 에 적용 불가능

MVC Matcher: 스프링 애플리케이션이 컨트롤러 동작에 대한 일치 요청을 이해하는 방법을 정확하게 지정.

MVC Matcher를 선택하는 것이 좋다. 스프링의 경로 및 작업 매핑과 관련한 몇가지 위험을 예방할 수 있다.

---
## Regex matcher

- `regexMatchers(HttpMethod method, String regex)`

  제한을 적용할 HTTP 방식과 경로를 참조할 정규식을 모두 지정.
  같은 경로 그룹에 대해 HTTP 방식별로 다른 제한을 적용할 때 유용.

- `regexMatchers(String regex)`
  
  경로만을 기준으로 권한 부여 제한을 적용할 때 이용 가능.
  모든 HTTP 방식에 자동으로 제한이 적용.

``` java
    protected void configure(HttpSecurity http) throws Exception {
        http.httpBasic();

        http.authorizeRequests()
                .regexMatchers(".*/(us|uk|ca)+/(en|fr).*")
                    .authenticated()
            .anyRequest().hasAuthority("premium");
    }
```

MVC와 Ant 식으로 문제를 해결하지 못했을 때만 정규식 사용.

```
(?:[a-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\.[a-z0-9!#$%&'*+/=?^_`{|}~-]+)*|"(?:[\x01- \x08\x0b\x0c\x0e-\x1f\x21\x23-\x5b\x5d-\x7f]|\\[\x01-\x09\x0b\x0c\x0e- \x7f])*")@(?:(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\.)+[a-z0-9](?:[a-z0-9- ]*[a-z0-9])?|\[(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0- 5]|2[0-4][0-9]|[01]?[0-9][0-9]?|[a-z0-9-]*[a-z0-9]:(?:[\x01- \x08\x0b\x0c\x0e-\x1f\x21-\x5a\x53-\x7f]|\\[\x01-\x09\x0b\x0c\x0e- \x7f])+)\])
```

---
## 요약
- 실제 시나리오에서는 요청마다 다른 권한 부여 규칙을 적용하는 경우가 많다.
- 경로와 HTTP 방식에 따라 권한 부여 규칙을 구성할 요청을 지정한다.
  이를 위해 MVC, Ant, Regex 세 가지 Matcher 메서드를 이용하는 방법을 배웠다.
- MVC와 Ant Matcher는 서로 비슷하며 일반적으로 이 중 하나를 선택해 권한 부여 제한을 적용할 요청을 지정할 수 있다.
- 요구사항이 MVC나 Ant 식으로 해결하기에 너무 복잡할 때는 이보다 강력한 정규식으로 구현할 수 있다.
