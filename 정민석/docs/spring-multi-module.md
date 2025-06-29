## 서론

현대의 소프트웨어 개발에서 애플리케이션의 복잡성은 지속적으로 증가하고 있다. 단일 모듈로 구성된 전통적인 모놀리식 아키텍처는 초기 개발 속도는 빠르지만, 프로젝트 규모가 커질수록 여러 한계점들이 드러나게 된다. 이러한 문제를 해결하기 위한 접근법 중 하나가 바로 멀티모듈 아키텍처이다.

본 글에서는 Spring Boot 기반의 서버 개발 과정에서 적용한 멀티모듈 아키텍처의 설계 원칙과 실제 구현 사례를 통해, 멀티모듈 아키텍처의 장단점과 적용 방법을 상세히 분석해보고자 한다.

## 멀티모듈 아키텍처란 무엇인가

### 정의와 개념

멀티모듈 아키텍처는 하나의 큰 애플리케이션을 기능적으로 독립적인 여러 모듈로 분리하여 구성하는 아키텍처 패턴이다. 각 모듈은 명확한 책임과 역할을 가지며, 모듈 간의 의존성은 명시적으로 정의된다.

### 모놀리식 vs 멀티모듈 구조 비교

전통적인 모놀리식 구조에서는 모든 컴포넌트가 하나의 모듈 내에 존재한다

```
monolithic-app/
├── src/main/java/
│   ├── controller/
│   ├── service/
│   ├── repository/
│   ├── entity/
│   └── config/
└── build.gradle
```

반면 멀티모듈 구조는 기능과 계층에 따라 분리된다

```
multi-module-app/
├── core/core-api/          # Presentation Layer
├── domain/                 # Business Layer
├── storage/db-core/        # Data Access Layer
├── admin/                  # Admin Interface
├── support/logging/        # Cross-cutting Concerns
└── clients/client-example/ # External Integration
```

## 실제 프로젝트 적용 사례

### 아키텍처 설계 원칙

멀티모듈 방식에서 다음과 같은 설계 원칙을 적용했다

1.  **레이어별 분리**: Presentation, Business, Data Access 계층을 물리적으로 분리
2.  **의존성 방향 제어**: 상위 계층이 하위 계층에만 의존하도록 구성
3.  **기술 독립성**: 비즈니스 로직이 특정 기술(JPA, Spring 등)에 종속되지 않도록 설계
4.  **단일 책임 원칙**: 각 모듈이 하나의 명확한 책임을 가지도록 구성

### 모듈 구조 상세 분석

#### 1\. core-api 모듈 (Presentation Layer)

```
// build.gradle.kts
dependencies {
    implementation(project(":domain"))
    implementation(project(":storage:db-core"))
    implementation(project(":admin"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
}
```

**역할과 책임**

-   HTTP 요청/응답 처리
-   인증 및 보안 설정
-   API 문서화
-   입력 값 검증

**주요 구성 요소**

-   Controller: REST API 엔드포인트 정의
-   Security: OAuth2, JWT 기반 인증
-   Configuration: 웹 관련 설정

#### 2\. domain 모듈 (Business Layer)

```
// build.gradle.kts
dependencies {
    compileOnly("org.springframework:spring-context")
    compileOnly("org.springframework:spring-tx")
    compileOnly("jakarta.transaction:jakarta.transaction-api")
}
```

**역할과 책임**

-   비즈니스 로직 구현
-   도메인 규칙 정의
-   서비스 인터페이스 제공
-   트랜잭션 관리

**핵심 설계 특징**

-   기술 종속성 최소화 (compileOnly 사용)
-   순수한 비즈니스 로직 집중
-   인터페이스 기반 설계

**도메인 구조**

```
domain/
├── user/           # 사용자 관리
├── course/         # 강의 관리
├── enrollment/     # 수강 관리
├── payment/        # 결제 처리
├── lecture/        # 강의 콘텐츠
└── review/         # 리뷰 시스템
```

#### 3\. storage/db-core 모듈 (Data Access Layer)

```
// build.gradle.kts
dependencies {
    implementation(project(":domain"))
    api("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("com.mysql:mysql-connector-j")
}
```

**역할과 책임**

-   데이터베이스 접근
-   JPA 엔티티 정의
-   Repository 구현
-   데이터 영속성 관리

**설계 특징**

-   도메인 객체와 JPA 엔티티 분리
-   Repository 패턴 구현
-   연관관계 매핑 최소화

### 의존성 관리 전략

#### Gradle 멀티모듈 설정

**settings.gradle.kts**

```
include(
    "core:core-api",
    ":admin",
    ":domain",
    "storage:db-core",
    "tests:api-docs",
    "support:logging",
    "support:monitoring",
    "clients:client-example",
)
```

**루트 build.gradle.kts 핵심 설정**

```
subprojects {
    apply(plugin = "org.jetbrains.kotlin.jvm")
    apply(plugin = "org.springframework.boot")

    dependencies {
        implementation("org.jetbrains.kotlin:kotlin-reflect")
        testImplementation("org.springframework.boot:spring-boot-starter-test")
    }

    tasks.getByName("bootJar") {
        enabled = false  // 기본적으로 비활성화
    }

    tasks.getByName("jar") {
        enabled = true   // 라이브러리 jar 활성화
    }
}
```

#### 의존성 방향 제어

```
core-api → domain → storage
    ↓
  admin
    ↓
 support modules
```

이러한 의존성 구조를 통해

-   순환 의존성 방지
-   계층별 책임 명확화
-   테스트 용이성 확보

## 멀티모듈 아키텍처의 장점

### 1\. 관심사의 분리 (Separation of Concerns)

각 모듈이 명확한 책임을 가짐으로써

-   코드의 가독성 향상
-   유지보수성 증대
-   버그 발생 시 영향 범위 제한

### 2\. 개발 생산성 향상

**병렬 개발 가능**

```
Team A: core-api 모듈 (API 개발)
Team B: domain 모듈 (비즈니스 로직)
Team C: storage 모듈 (데이터 계층)
```

**부분 빌드 지원**

```
# 특정 모듈만 빌드
./gradlew :domain:build

# 의존성이 있는 모듈들만 빌드
./gradlew :core-api:build
```

### 3\. 테스트 격리

```
// domain 모듈 단위 테스트 - 기술 종속성 없음
@Test
fun `사용자 등록 시 이메일 중복 검증`() {
    val userService = UserService(mockUserRepository)

    assertThrows<DuplicateEmailException> {
        userService.register("existing@email.com")
    }
}

// storage 모듈 통합 테스트 - JPA 관련 테스트
@DataJpaTest
class UserRepositoryTest {
    @Test
    fun `이메일로 사용자 조회`() {
        // JPA 관련 테스트 로직
    }
}
```

### 4\. 기술 스택 유연성

모듈별로 다른 기술 스택 적용 가능

-   core-api: Spring WebMVC
-   domain: 순수 Kotlin/Java
-   storage: JPA, MyBatis 선택 가능
-   clients: WebClient, Feign 등 선택 가능

### 5\. 배포 전략의 다양성

```
// core-api만 실행 가능한 JAR로 빌드
// core-api/build.gradle.kts
tasks.getByName("bootJar") {
    enabled = true
}

// 다른 모듈들은 라이브러리로 사용
tasks.getByName("jar") {
    enabled = true
}
```

## 멀티모듈 아키텍처의 단점과 한계

### 1\. 초기 설정의 복잡성

**Gradle 설정 복잡도 증가**

-   모듈 간 의존성 관리
-   버전 관리 복잡성
-   빌드 스크립트 중복

**해결 방안**

```
// 루트 build.gradle.kts에서 공통 설정 관리
subprojects {
    // 공통 의존성 및 설정
}

// gradle.properties에서 버전 중앙 관리
springBootVersion=3.2.0
kotlinVersion=1.9.20
```

### 2\. 개발 초기 오버헤드

**모놀리식 대비 초기 개발 속도 저하**

-   모듈 구조 설계 시간 필요
-   인터페이스 정의 오버헤드
-   모듈 간 통신 코드 작성

### 3\. 디버깅의 복잡성

**스택 트레이스 추적 어려움**

```
core-api → domain → storage
```

여러 모듈을 거치는 호출 스택 추적 시 복잡성 증가

**해결 방안**

-   적절한 로깅 전략 수립
-   통합 테스트 강화
-   모니터링 도구 활용

### 4\. 순환 의존성 위험

잘못된 설계 시 순환 의존성 발생 가능

```
domain → storage → domain (X)
```

**방지 전략**

-   명확한 계층 구조 정의
-   의존성 방향 규칙 수립
-   정기적인 의존성 분석

## 모놀리식 vs 멀티모듈 트레이드오프 분석

### 개발 속도 관점

| 구분 | 모놀리식 | 멀티모듈 |
| --- | --- | --- |
| 초기 개발 | 빠름 (프로토타입에 유리) | 느림 (설계 시간 필요) |
| 중장기 개발 | 점진적 저하 | 안정적 유지 |
| 팀 확장성 | 제한적 | 우수함 |

### 유지보수성 관점

**모놀리식의 문제점**

```
// 모든 것이 하나의 패키지에 혼재
com.example.app
├── UserController.java
├── UserService.java
├── UserRepository.java
├── CourseController.java
├── CourseService.java
├── PaymentController.java
└── PaymentService.java
```

**멀티모듈의 이점**

```
domain/user/
├── User.java
├── UserService.java
└── UserRepository.java

domain/course/
├── Course.java
├── CourseService.java
└── CourseRepository.java
```

### 성능 관점

**빌드 성능**

-   모놀리식: 전체 재빌드 필요
-   멀티모듈: 변경된 모듈만 빌드 가능

**런타임 성능**

-   모놀리식: 단일 JVM, 메모리 효율적
-   멀티모듈: 동일한 JVM 내 실행, 성능 차이 미미

### 테스트 관점

**모놀리식**

```
@SpringBootTest  // 전체 컨텍스트 로딩
class UserServiceTest {
    // 무거운 통합 테스트
}
```

**멀티모듈**

```
// domain 모듈 - 경량 단위 테스트
class UserServiceTest {
    // 순수 비즈니스 로직 테스트
}

// storage 모듈 - 데이터 계층만 테스트
@DataJpaTest
class UserRepositoryTest {
    // JPA 관련 테스트만
}
```

## 실제 구현 시 고려사항

### 1\. 모듈 분리 기준

**도메인 기반 분리**

```
domain/
├── user/          # 사용자 관련 비즈니스 로직
├── course/        # 강의 관련 비즈니스 로직
├── enrollment/    # 수강 관련 비즈니스 로직
└── payment/       # 결제 관련 비즈니스 로직
```

**계층 기반 분리**

```
core-api/          # Presentation Layer
domain/            # Business Layer
storage/           # Data Access Layer
```

### 2\. 모듈 간 통신 전략

**직접 의존성**

```
// core-api에서 domain 직접 호출
@RestController
class UserController(
    private val userService: UserService
) {
    @PostMapping("/users")
    fun createUser(@RequestBody request: CreateUserRequest) {
        return userService.createUser(request.toCommand())
    }
}
```

**이벤트 기반 통신**

```
// 도메인 이벤트 발행
@Service
class EnrollmentService(
    private val eventPublisher: ApplicationEventPublisher
) {
    fun enroll(userId: Long, courseId: Long) {
        // 수강 등록 로직
        eventPublisher.publishEvent(EnrollmentCompletedEvent(userId, courseId))
    }
}

// 이벤트 처리
@EventListener
class PaymentEventHandler {
    fun handle(event: EnrollmentCompletedEvent) {
        // 결제 처리 로직
    }
}
```

### 3\. 공통 코드 관리

**support 모듈 활용**

```
support/
├── logging/       # 로깅 설정
├── monitoring/    # 메트릭, 헬스체크
└── common/        # 공통 유틸리티
```

**버전 관리 전략**

```
// gradle.properties
applicationVersion=1.0.0
springBootVersion=3.2.0

// 모든 모듈이 동일한 버전 사용
version = "${property("applicationVersion")}"
```

## 결론

멀티모듈 아키텍처는 복잡한 애플리케이션 개발에서 코드 품질과 유지보수성을 크게 향상시킬 수 있는 강력한 도구이다. 특히 팀 규모가 크거나 장기적인 프로젝트에서 그 진가를 발휘한다.

### 적용 권장 시나리오

**멀티모듈을 선택해야 하는 경우**

-   팀 규모가 5명 이상인 프로젝트
-   장기간 유지보수가 예상되는 시스템
-   도메인 복잡도가 높은 비즈니스 애플리케이션
-   여러 팀이 협업하는 환경

**모놀리식을 유지하는 것이 나은 경우**

-   프로토타입 또는 MVP 개발
-   팀 규모가 작은 프로젝트 (3명 이하)
-   단순한 CRUD 애플리케이션
-   빠른 출시가 중요한 프로젝트

### 마이그레이션 전략

기존 모놀리식에서 멀티모듈로 전환 시

1.  **점진적 분리**: 한 번에 모든 것을 분리하지 말고 단계적으로 진행
2.  **테스트 커버리지 확보**: 분리 전 충분한 테스트 코드 작성
3.  **의존성 분석**: 기존 코드의 의존성 관계 파악
4.  **팀 교육**: 멀티모듈 개발 방식에 대한 팀 교육 실시

멀티모듈 아키텍처는 초기 투자 비용이 있지만, 장기적으로는 개발 생산성과 코드 품질 향상이라는 큰 이익을 가져다준다. 프로젝트의 특성과 팀의 상황을 종합적으로 고려하여 적절한 아키텍처를 선택하는 것이 중요하다.
