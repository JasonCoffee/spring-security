# 7장 권한 부여 구성: 액세스 제한

### 권한 부여 (Authorization)

- 식별된 클라이언트가 요청된 리소스에 액세스할 권한이 있는지 시스템이 결정하는 프로세스.
- 애플리케이션에서는 식별된 모든 사용자가 시스템의 모든 리소스에 액세스할 수 있는 것은 아니다.
- 사용자는 가진 권한에 따라 특정 작업만 실행할 수 있다.
- 권한 부여는 항상 인증 이후에 수행.


### 인증 (Authentication)
애플리케이션이 리소스의 호출자를 식별하는 프로세스.

- - -
```
스프링 시큐리티에서는 애플리케이션은 인증 흐름을 완료한 후 요청을 권한 부여 필터에 위임한다.
인증 필터 → 권한 부여 필터
필터는 구성된 권한 부여 규칙에 따라 요청을 허용하거나 거부한다.

GrantedAuthority → 권한 부여 프로세스와 연관
```
- - -
### 권한에 대한 접근 구성 방법

#### hasAuthority()
하나의 권한만 매개변수로 가능
``` java
http.authorizeRequests().anyRequest().hasAuthority("WRITE");
```
→ 엔드포인트를 호출하려면 사용자에게 `WRITE` 권한이 필요하다고 지정하는 것.

#### hasAnyAuthority()
하나 이상의 권한을 부여
``` java
http.authorizeRequests().anyRequest().hasAnyAuthority("WRITE", "READ");
```

#### access()
SpEL 기반으로 권한 부여 규칙 부여
``` java
http.authorizeRequests().anyRequest().access("hasAuthority('WRITE')");

## 사용자에게 읽기 권한이 있어야 하지만 삭제 권한은 없어야 함을 의미.
http.authorizeRequests().anyRequest()
        .access("hasAuthority('read') and !hasAuthority('delete')");
```

- - -
### 역할에 대한 접근 구성 방법

- 역할은 사용자가 수행할 수 있는 작업을 나타내는 다른 방법이다.
- 역할과 권한의 차이점을 이해하는 것이 필요.
```
권한 → 결이 고운 (fine grained)
역할 → 결이 굵은 (coarse grained)

authorities("ROLE_ADMIN")
roles("ADMIN")
```
- - -
#### hasRole()
애플리케이션이 요청을 승인할 하나의 역할 이름을 매개변수로 받는다.
``` java
http.authorizeRequests().anyRequest().hasRole("ADMIN");
```

#### hasAnyRole()
애플리케이션이 요청을 승인할 여러 역할 이름을 매개변수로 받는다.
``` java
http.authorizeRequests().anyRequest().hasAnyRole("ADMIN", "MANAGER");
```

#### access()
애플리케이션이 요청을 승인할 역할을 SpEL 방식으로 받는다.
``` java
http.authorizeRequests()
        .anyRequest().access("T(java.time.LocalTime).now().isAfter(T(java.time.LocalTime).of(12, 0))");
```
- - -

#### permitAll()
모든 사용자의 접근을 허용
``` java
http.authorizeRequests().anyRequest().permitAll();
```

#### denyAll()
모든 사용자의 접근을 제한
``` java
http.authorizeRequests().anyRequest().denyAll();
```

- - -
### 요약
- 권한 부여는 애플리케이션이 인증된 요청을 허가할지 결정하는 프로세스다. 권한 부여는 항상 인증 이후에 수행된다.
- 애플리케이션이 인증된 사용자의 권한과 역할에 따라 권한을 부여하는 방법을 구성할 수 있다.
- 애플리케이션에서 인증되지 않은 사용자가 특정 요청을 수행할 수 있게 지정할 수도 있다.
- denyAll() 메서드로 앱이 모든 요청을 거부하고, permitAll() 메서드로 모든 요청을 수락하게 할 수 있다.
