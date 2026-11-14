---
title: "Spring 테스트에 진짜 DB를 가져오는 법, Testcontainers"
description: "가짜 DB 대신 진짜 DB로 테스트하고 싶을 때. 비개발자도 이해할 수 있는 비유로 Testcontainers를 설명합니다."
pubDatetime: 2026-11-01T12:00:00+09:00
tags:
  - 개발
  - Spring
  - Testcontainers
---

개발자는 코드를 짜고 나면 "이게 정말 잘 동작하나?"를 확인하는 **테스트 코드**를 함께 작성합니다. 그런데 테스트를 하다 보면 꼭 부딪히는 질문이 하나 있습니다.

> 테스트할 때 데이터베이스는 뭘 써야 하지?

이 글에서는 그 질문에 대한 요즘의 정답 중 하나인 **Testcontainers**를 소개합니다. 개발자가 아니어도 읽을 수 있도록 식당 비유를 곁들여 설명해 보겠습니다.

## 왜 Testcontainers를 쓸까?

**1. 왜 Testcontainers를 쓰나요?**
테스트할 때마다 **실제 서비스와 똑같은 DB**를, 아무도 건드리지 않은 **깨끗한 상태로** 쓰고 싶어서입니다.

**2. 왜 실제와 똑같은 DB여야 하나요?**
흔히 쓰던 가짜 DB(H2)에서 통과한 테스트가, 실제 서비스의 DB(PostgreSQL 등)에서는 실패하는 일이 생기기 때문입니다. DB마다 지원하는 기능과 동작 방식이 조금씩 다르고, 가짜 DB는 실제 DB를 "흉내"만 낼 뿐입니다.

**3. 그럼 왜 예전에는 진짜 DB로 테스트하지 않았나요?**
진짜 DB를 테스트용으로 준비하고 치우는 일이 **너무 번거로웠기** 때문입니다. 개발자마다 DB를 직접 설치하거나, 팀이 공용 DB 하나를 같이 쓰다가 서로의 데이터가 엉키곤 했습니다. 그래서 많은 팀이 "실제와 조금 달라도 편한" 가짜 DB로 타협했습니다.

**4. Docker가 나온 뒤에도 문제가 남았나요?**
Docker 덕분에 진짜 DB를 쉽게 띄울 수 있게 됐지만, **테스트 코드와 따로 놀았습니다.** 테스트 전에 사람이 직접 DB를 켜고, 끝나면 꺼야 했습니다. 깜빡하면 지난 테스트의 데이터가 남거나, 다른 프로그램과 포트가 겹쳐 실패했습니다. Testcontainers는 바로 **이 간극을 메우기 위해** Java 라이브러리로 탄생했습니다. "테스트 코드가 스스로 진짜 DB를 켜고, 끝나면 스스로 치우게 하자"는 아이디어입니다.

**5. 왜 그렇게까지 해서 테스트를 믿을 수 있어야 하나요?**
테스트를 믿을 수 있어야 개발자가 코드를 **겁내지 않고 빠르게 고칠 수 있기** 때문입니다. 테스트를 믿지 못하면 고칠 때마다 사람이 일일이 다시 확인해야 하고, 확인이 빠진 곳에서 사용자에게 오류가 터집니다.

> 🎯 **사용하는 이유:** 실제와 똑같은 환경에서 믿을 수 있는 테스트를 해야, 빠르고 안전하게 서비스를 바꿔 나갈 수 있습니다.
>
> 🎯 **극복한 문제:** "진짜 DB는 준비가 번거롭고, 가짜 DB는 믿을 수 없다"는 오랜 딜레마를, 테스트 코드가 진짜 DB를 직접 켜고 치우게 만들어 해결했습니다.

## 먼저, 테스트란?

> 🍳 **비유: 신메뉴 시식회**
>
> 식당에서 신메뉴를 손님에게 내기 전에, 주방에서 직접 만들어 맛을 봅니다. 레시피대로 만들었을 때 정말 그 맛이 나는지 확인하는 거죠.

소프트웨어의 테스트도 같습니다. 서비스를 사용자에게 내놓기 전에 "회원가입을 하면 정보가 잘 저장되는가?", "주문을 취소하면 재고가 다시 늘어나는가?"를 코드로 미리 확인합니다.

그런데 이런 기능은 대부분 **데이터베이스(DB)**, 즉 정보를 저장해 두는 창고를 사용합니다. 그래서 테스트할 때도 DB가 필요합니다. 문제는 **어떤 DB를** 쓰느냐입니다.

## 테스트용 DB를 준비하는 세 가지 방법

### 1. 다 같이 쓰는 공용 DB

팀 전체가 하나의 개발용 DB 서버를 함께 쓰는 방법입니다.

> 🍳 **비유: 여러 요리사가 함께 쓰는 공용 주방**
>
> 내가 시식용 파스타를 만들려는데, 옆 요리사가 내 냄비에 소금을 넣어버립니다. 맛이 이상한 게 내 레시피 탓인지, 옆 사람 탓인지 알 수가 없습니다.

다른 사람의 테스트가 남긴 데이터 때문에 내 테스트가 실패하기도 하고, 반대로 내가 다른 사람의 테스트를 망치기도 합니다. 서버가 꺼져 있으면 아무도 테스트를 못 합니다.

### 2. 메모리 안의 가짜 DB (H2)

테스트할 때만 프로그램 메모리 안에 작은 DB를 띄우는 방법입니다. Spring에서는 H2라는 DB가 자주 쓰입니다.

> 🍳 **비유: 실습용 모형 주방**
>
> 요리학원의 실습 주방은 깨끗하고 언제든 쓸 수 있습니다. 그런데 실제 가게 주방과 화구 세기도, 오븐 종류도 다릅니다. 실습 주방에서 완벽했던 요리가 가게에서는 타버릴 수 있습니다.

빠르고 편하지만, 실제 서비스에서 쓰는 DB(PostgreSQL, MySQL 등)와 **미묘하게 다르게 동작**합니다. PostgreSQL에만 있는 기능은 H2에서 테스트할 수 없고, H2에서 통과한 테스트가 실제 환경에서 실패하는 일이 생깁니다.

### 3. 테스트할 때마다 진짜 DB를 잠깐 빌려오기 (Testcontainers)

> 🍳 **비유: 시식할 때마다 똑같은 주방을 통째로 배달받기**
>
> 시식회를 할 때마다 실제 가게와 **똑같은 장비가 갖춰진 주방**이 트럭에 실려 옵니다. 깨끗한 상태로 도착하고, 나만 씁니다. 시식이 끝나면 트럭이 주방을 통째로 싣고 가버립니다. 설거지할 필요도 없습니다.

이것이 Testcontainers의 아이디어입니다. 테스트가 시작되면 **진짜 PostgreSQL**을 띄우고, 테스트가 끝나면 깨끗하게 치웁니다. 공용 주방처럼 남과 엉킬 일도 없고, 모형 주방처럼 실제와 다를 일도 없습니다.

| 방법          | 실제와 같은가?  | 남과 엉키지 않는가? | 준비가 쉬운가?  |
| ------------- | --------------- | ------------------- | --------------- |
| 공용 DB       | ✅              | ❌                  | ⚠️ 서버 필요    |
| H2 (가짜 DB)  | ❌              | ✅                  | ✅              |
| Testcontainers | ✅              | ✅                  | ✅ Docker만 있으면 |

## 잠깐, 컨테이너가 뭔가요?

Testcontainers라는 이름의 "컨테이너"는 **Docker 컨테이너**를 말합니다.

> 📦 **비유: 밀키트**
>
> 밀키트 상자 안에는 요리에 필요한 재료와 소스, 레시피가 전부 들어 있습니다. 누구 집 부엌에서 열든 똑같은 요리가 나옵니다.

Docker 컨테이너도 프로그램 하나가 돌아가는 데 필요한 모든 것을 상자 하나에 담아둔 것입니다. "PostgreSQL 17" 상자를 열면, 내 노트북이든 회사 서버든 **어디서나 똑같은 PostgreSQL 17**이 실행됩니다.

Testcontainers는 이 상자를 **테스트 코드에서 직접 열고 닫을 수 있게 해주는 도구**입니다.

## Testcontainers가 일하는 순서

1. **테스트 시작**: Testcontainers가 Docker에게 "PostgreSQL 상자 하나 열어줘"라고 부탁합니다.
2. **준비 완료 대기**: DB가 완전히 켜져서 요청을 받을 수 있을 때까지 기다립니다.
3. **연결 정보 전달**: 새로 뜬 DB의 주소, 아이디, 비밀번호를 Spring 애플리케이션에 알려줍니다.
4. **테스트 실행**: 테스트가 진짜 DB를 상대로 돌아갑니다.
5. **정리**: 테스트가 끝나면 컨테이너를 지웁니다.

> 🧹 **비유: 행사장 철거 담당 직원**
>
> 혹시 테스트가 중간에 갑자기 멈춰도 괜찮습니다. Testcontainers는 **Ryuk**이라는 철거 담당 컨테이너를 함께 띄워둡니다. 행사가 비정상적으로 끝나도 이 직원이 남은 주방을 확실히 치워줍니다.

## Spring Boot에서 써보기

이제부터는 개발자를 위한 실전 코드입니다. Spring Boot 4.1, Testcontainers 2.0 기준입니다.

### 준비물

- **Docker**: 내 컴퓨터에 Docker Desktop(또는 OrbStack, Colima 등)이 실행 중이어야 합니다.
- **의존성 추가** (`build.gradle.kts`)

```kotlin
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.boot:spring-boot-testcontainers")
    testImplementation("org.testcontainers:testcontainers-junit-jupiter")
    testImplementation("org.testcontainers:testcontainers-postgresql")
}
```

버전은 Spring Boot가 관리해 주므로 적지 않아도 됩니다.

> 💡 Testcontainers 2.0부터 모듈 이름에 `testcontainers-` 접두사가 붙었습니다. 예전 글에서 보이는 `org.testcontainers:postgresql`은 1.x 시절 이름입니다.

### 가장 기본적인 테스트

```kotlin
import org.springframework.boot.testcontainers.service.connection.ServiceConnection
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers
import org.testcontainers.postgresql.PostgreSQLContainer

@SpringBootTest
@Testcontainers
class MemberRepositoryTest(
    @Autowired private val memberRepository: MemberRepository,
) {

    @Test
    fun `회원을 저장하면 다시 조회할 수 있다`() {
        val saved = memberRepository.save(Member("원형"))

        assertThat(memberRepository.findById(saved.id)).isPresent
    }

    companion object {
        @Container
        @ServiceConnection
        @JvmStatic
        val postgres = PostgreSQLContainer("postgres:17-alpine")
    }
}
```

> 💡 Kotlin에는 Java의 `static`이 없어서 `companion object`에 `@JvmStatic`을 붙여 같은 효과를 냅니다. 테스트 클래스 전체가 공유하는 컨테이너를 선언할 때 쓰는 관용구입니다.

어노테이션 세 개가 핵심입니다.

| 어노테이션 | 하는 일 | 식당 비유 |
| --- | --- | --- |
| `@Testcontainers` | 이 테스트 클래스에서 컨테이너를 관리하겠다고 선언 | "오늘 시식회는 배달 주방을 씁니다" |
| `@Container` | 테스트 전에 컨테이너를 켜고, 끝나면 끔 | 주방 트럭을 부르고 돌려보내기 |
| `@ServiceConnection` | 컨테이너의 접속 정보를 Spring에 자동으로 연결 | 새 주방 주소를 요리사에게 알려주기 |

`application.yml`에 DB 주소를 따로 적을 필요가 없습니다. 컨테이너는 실행할 때마다 빈 포트를 골라 뜨기 때문에 주소가 매번 바뀌는데, `@ServiceConnection`이 그 주소를 알아서 Spring에 넘겨줍니다.

### 예전 방식: `@DynamicPropertySource`

Spring Boot 3.1 이전에는 접속 정보를 직접 넘겨줘야 했습니다. 오래된 프로젝트에서 이런 코드를 만나면 같은 일을 하는 코드라고 이해하면 됩니다.

```kotlin
companion object {
    @JvmStatic
    @DynamicPropertySource
    fun configureProperties(registry: DynamicPropertyRegistry) {
        registry.add("spring.datasource.url") { postgres.jdbcUrl }
        registry.add("spring.datasource.username") { postgres.username }
        registry.add("spring.datasource.password") { postgres.password }
    }
}
```

## 테스트가 느려졌다면: 컨테이너 함께 쓰기

위 방식은 테스트 **클래스마다** 컨테이너를 새로 띄웁니다. 테스트 클래스가 수십 개면 DB도 수십 번 켜졌다 꺼집니다.

> 🍳 **비유: 손님 한 팀마다 주방 트럭을 새로 부르기**
>
> 시식회에 손님이 열 팀 오는데, 팀마다 트럭을 새로 부르면 하루가 다 갑니다. 그날은 트럭 한 대를 세워두고 팀이 바뀔 때마다 테이블만 정리하는 게 효율적입니다.

공용 설정 클래스를 만들어 컨테이너를 **Spring Bean**으로 등록하면, 여러 테스트 클래스가 하나의 컨테이너를 함께 씁니다.

```kotlin
@TestConfiguration(proxyBeanMethods = false)
class TestcontainersConfig {

    @Bean
    @ServiceConnection
    fun postgres(): PostgreSQLContainer = PostgreSQLContainer("postgres:17-alpine")
}
```

```kotlin
@SpringBootTest
@Import(TestcontainersConfig::class)
class MemberRepositoryTest {
    // 컨테이너 선언 없이 바로 테스트 작성
}
```

Spring은 같은 설정을 쓰는 테스트들끼리 애플리케이션 컨텍스트를 재사용합니다. 그래서 컨테이너도 한 번만 뜹니다. 대신 여러 테스트가 같은 DB를 쓰게 되므로, 각 테스트가 끝날 때 데이터를 정리해야 합니다. `@Transactional`로 롤백하거나 테스트마다 데이터를 지우세요.

## 덤: 로컬 개발 서버에도 쓰기

Testcontainers는 테스트 전용이 아닙니다. 위 설정 클래스를 재사용하면 **내 컴퓨터에 DB를 설치하지 않고도** 애플리케이션을 실행할 수 있습니다.

`src/test/kotlin`에 아래 파일을 만들고 실행하면, PostgreSQL 컨테이너가 함께 뜬 상태로 애플리케이션이 시작됩니다.

```kotlin
fun main(args: Array<String>) {
    fromApplication<Application>()
        .with(TestcontainersConfig::class.java)
        .run(*args)
}
```

> 🍳 **비유: 개업 전 리허설**
>
> 새로 들어온 직원이 첫날 "주방 장비 설치부터 하세요"라는 말을 듣지 않아도 됩니다. 버튼 하나면 똑같은 주방이 준비됩니다.

## 알아둘 점

- **Docker가 꼭 필요합니다.** Docker가 꺼져 있으면 테스트가 바로 실패합니다. CI(GitHub Actions 등)에서도 Docker를 쓸 수 있어야 합니다. GitHub Actions의 Ubuntu 러너에는 기본으로 설치되어 있습니다.
- **첫 실행은 느립니다.** 처음에는 PostgreSQL 이미지를 내려받아야 해서 시간이 걸립니다. 두 번째부터는 내려받아 둔 이미지를 씁니다.
- **H2보다는 느립니다.** 진짜 DB를 켜는 데 몇 초가 걸립니다. 위의 "컨테이너 함께 쓰기"로 대부분 해결됩니다.
- **이미지 버전은 운영 환경과 맞추세요.** 운영에서 PostgreSQL 16을 쓴다면 테스트도 `postgres:16`으로 맞춰야 의미가 있습니다.

## 정리

> 🍳 테스트할 때마다 **실제 가게와 똑같은 주방**을 배달받아 쓰고, 끝나면 통째로 돌려보낸다.

이 한 줄이 Testcontainers의 전부입니다. 남과 엉키지 않고, 실제와 다르지 않은 환경에서 테스트할 수 있습니다. Spring Boot에서는 `@ServiceConnection` 덕분에 설정도 거의 필요 없습니다.

PostgreSQL 말고도 MySQL, Redis, Kafka, Elasticsearch 등 대부분의 인프라를 같은 방식으로 띄울 수 있습니다. "이건 테스트하기 어려워서 못 했어"라고 미뤄둔 부분이 있다면, Testcontainers로 다시 도전해 보세요.
