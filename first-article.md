## Intro  
![image](https://github.com/gukin-han/article-draft/assets/115940366/5d1565dd-d9de-49f4-91ce-786f74d14e1d)
![image](https://github.com/gukin-han/article-draft/assets/115940366/be4f75fe-7807-43b8-9101-ebeef142cb53)  

현재 위와 같은 아키텍처를 구상하고 구현 중인데 애플리케이션의 시간 정보와 시간대를 어떻게 처리해야하는지에 대한 고민이 생겼다. 백엔드에서 로컬 타임으로 DB에 저장하니 데이터는 KTC를 따르지만, DB의 시간대는 UTC를 나타낸다.  

이러한 시간대 불일치는 프로덕트에서 큰 에러나 장애 요소로 작용할거라고 예상이 되는데 실제로 주변 사람이 서로 다른 시간대로 구성된 서버들에서 발견되는 문제들로 골머리를 썩히고 있는 모습을 보았다. 처음에는 시간대 불일치를 가벼운 함수로 처리를 했겠지만, 애플리케이션이 복잡해지면서 어떤 경로를 타고 시간 처리를 하는지 생각하기 어렵기 때문이다.  

예약 시스템과 같이 시간이 중요한 서비스에서는 굉장히 중요한 주제라고 생각해왔는데 지금까지 이것을 다룬 레퍼런스(책과 강의)를 본적이 없는것 같다. 따라서, 몇가지 지켜야할 컨벤션과 프론트 및 백엔드에서의 설정들을 정리해두고 지켜보려고 한다.  

## 서버에서 시간 정보와 시간대 처리  
![image](https://github.com/gukin-han/article-draft/assets/115940366/c3ae4399-1947-453b-af66-134028a16e00)  

현재 아키텍처를 단순화하면 위 그림과 같다. 백엔드서버와 DB서버는 UTC를 사용하는것이 Best Practice라고 하는데 그 이유는 다음과 같다:

* 코드의 단순화
* 버그 및 오류 감소
* 데이터 일관성

따라서, 백엔드 서버와 DB 서버는 UTC를 사용하게 되면 변환에 대한 코드가 백엔드 소스 코드에 작성될 일이 없다. 현재의 아키텍처도 레이어드 아키텍처를 어느 정도 따르는데, 특정 로직을 처리하는 계층을 최끝단으로 옮겨두는 방식이 가장 효율적이라고 생각한다.

단순히 리액트UI 라이브러리를 고려한다면 여기까지 생각할 수 있지만 Next 처럼 RSC를 사용한다면 좀더 고민거리가 생긴다.

## Next 서버를 고려한다면
![image](https://github.com/gukin-han/article-draft/assets/115940366/ea49469b-12a8-4e69-ad14-7d54f618f21c)
Next 프레임워크를 사용한다면 Next의 서버도 같이 고려해줘야 한다. 위에서 단순화한 그림을 Next 관점에서 세분화해주면 클라이언트 컴포넌트, 서버액션, 서버 컴포넌트 세 가지로 나눠질 수 있다. Next의 렌더링 방식을 이해할 필요가 있는데, 먼저 유저의 요청에 따라 서버 컴포넌트와 클라이언트 컴포넌트가 한 번 렌더링을 해서 브라우저로 전달이 된다. 그리고 클라이언트 컴포넌트에서 JS를 추출해서 한 번 더 브라우저로 던져지는데 이때 클라이언트 컴포넌트가 제대로 렌더링이 된다.

### 서버에서 브라우저로
ISR, SSR까지 고려했을때 시간에 대한 모든 데이터 변환 로직은 클라이언트 컴포넌트에서 하는게 맞다. 즉, Next 서버, 백엔드 서버, DB서버는 모두 UTC로 처리하는게 편리하고 브라우저로 전달되는 모든 데이터는 UTC라고 생각한다. 그렇다면 전달된 UTC는 브라우저가 있는 로컬 타임으로 변환시키는것이 합리적이다. 이때 JS가 필요하기 때문에 클라이언트 컴포넌트에서만 시간 관리를 한다.

### 브라우저에서 서버로
Next에서 브라우저에서 서버로 가기 위해서는 서버액션을 이용한 방법이 앞으로 대세가 될것으로 예상된다. 서버액션은 클라이언트 컴포넌트와 서버 컴포넌트의 중간 매개체 느낌인데, 브라우저에서 발생한 데이터를 담아서 서버에서 실행되게끔 한다. 즉, 브라우저에서 HTTP POST 요청을 통해 서버로 이동해서 대신 서버가 로직 처리를 한다.

따라서, 여기서는 두 가지 방법이 있을것 같은데 첫 번째 방법은 브라우저에서 폼 데이터를 담을때 로컬에서 UTC로 변환하는 작업을 수행해서 항상 서버로 UTC를 넘겨주는 방법이다. 두 번째 방법은 지역 정보를 담아서 로컬 시간을 넘겨주고 Next 서버에서 다시 변환을 하는 작업이다. 두 가지를 비교했을때 첫 번째 방법이 가장 깔끔하게 처리할 수 있을것으로 보인다.

## Spring에서 시간 설정
![image](https://github.com/gukin-han/article-draft/assets/115940366/886b402c-cf8d-40ef-9aa7-84404b1ce8ac)
가장 간단한 방법은 JVM에 시간 설정을 하는 방법이다 \[[1](https://www.baeldung.com/java-jvm-time-zone), [2](https://stackoverflow.com/questions/54316667/how-do-i-force-a-spring-boot-jvm-into-utc-time-zone), [3](https://www.onlinetutorialspoint.com/spring-boot/how-to-set-spring-boot-settimezone.html)\]. 메인 클래스에 @PostConstruct 어노테이션이 붙은 메서드에 시간을 설정하는 로직을 작성하면 된다. 예제 코드는 아래와 같다.

### 프로덕션 환경

**MyrealogApplication.java**

```
@EnableJpaAuditing
@SpringBootApplication
public class MyrealogApplication {

	public static void main(String[] args) {
		SpringApplication.run(MyrealogApplication.class, args);
	}

	@PostConstruct
	public void init() {
		TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
	}
}
```

위 코드는 JVM에서의 시간 설정이므로 DB 커넥션에도 설정을 추가로 필요하다. 아무리 DB에 UTC가 되어있더라도 DB 커넥션에 시간 설정을 하지 않으면 JPA를 통해 DB로 들어가는 시간 데이터는 로컬 시간으로 들어가는 것을 확인했다.

**application.yml**

```
# 기본 설정
spring:
  datasource:
    url: jdbc:mysql:${RDS_ENDPOINT}?serverTimezone=UTC
    username: ${USERNAME}
    password: ${PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
```

MySQL 기준으로 위와 같이 작성하면 DB로 들어가는 시간 데이터가 UTC로 저장된다. 또한, 대부분의 DB의 디폴트 시간존은 UTC이기 때문에 디폴트를 그대로 따르는게 좋다.

### 테스트 환경

테스트 환경에서는 동일하게 JVM을 위와 같이 설정하면 된다. 만약 h2를 테스트 환경에서 사용한다면 아래와 같은 세팅을 해주면 될것 같다. 아직 테스트를 안해보았지만 관련 h2 문서를 레퍼런스로 남겨둔다 \[[4](https://www.h2database.com/html/commands.html#set_time_zone)\].

**application.yml**

```
---
spring:
  datasource:
    url: jdbc:h2:tcp://localhost/~/test;TIME ZONE=UTC
    username: sa
    password:
    driver-class-name: org.h2.Driver
```
![image](https://github.com/gukin-han/article-draft/assets/115940366/5f0dd39e-d2e9-450e-8c9f-74bdbe22da72)  

이를 콘솔에서 확인하려면 콘솔도 위 처럼 설정을 하고 들어가야 한다. 그리고 콘솔에서 select now() 쿼리를 날려보면 UTC 시간을 보여준다. 위 상태로 데이터 베이스에 시간 데이터를 밀어넣어주면 아래 처럼 UTC가 적용되는것을 확인할 수 있다.  

![image](https://github.com/gukin-han/article-draft/assets/115940366/f36bcae5-655d-47b5-8c2d-106f5668577f)
![image](https://github.com/gukin-han/article-draft/assets/115940366/b0c93210-69a5-4653-adbd-3c231ccb8233)

## Outro

이미 인프라가 구축된 환경에서 일하는 대기업 직원들은 크게 신경쓸일이 없을것 같고, 막 구축을 해서 나아가는 스타트업(=중소기업)들은 이러한 이슈들을 미리 생각해두고 설정을 해둬야 큰 문제를 예방할 수 있지 않을까?

## References

1.  [J. Cook (2024) Baeldung "How to Set the JVM Time Zone"](https://www.baeldung.com/java-jvm-time-zone).
2.  [Y. Kim (2019) StackOverflow "How do I force a Spring Boot JVM into UTC time zone?"](https://stackoverflow.com/questions/54316667/how-do-i-force-a-spring-boot-jvm-into-utc-time-zone).
3.  [Online Tutorials  Point (2018) "How to set Spring Boot SetTimeZone"](https://www.onlinetutorialspoint.com/spring-boot/how-to-set-spring-boot-settimezone.html).
4.  [h2 "set time zone"](https://www.h2database.com/html/commands.html#set_time_zone).
