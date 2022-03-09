---
title: "Spring REST Docs 찍먹하기"
date: 2022-03-08T21:25:54+09:00
draft: true
---

# Spring REST Docs 찍먹하기

Spring Webflux + Kotlin + Gradle(Kotlin DSL) + JUnit 5

Spring REST Docs의 레퍼런스 가이드는 [여기](https://docs.spring.io/spring-restdocs/docs/current/reference/html5/).

이제까지 문서화는 Swagger 라고 알고 있었는데 프로덕션 코드에 어노테이션을 덕지덕지 붙이는게 싫어서 검색을 좀 해보았습니다. Spring REST Docs라는 것을 찾았는데, 문서화를 하기 위해서 테스트를 작성해야한다는 컨셉이 마음에 들었습니다. 적용하는 과정에서 은근히 삽질도 많이 하였기에 기록용으로 남겨봅니다.

포스트를 작성하면서 참고한 [샘플](https://github.com/spring-projects/spring-restdocs/tree/v2.0.6.RELEASE/samples/web-test-client)

### 시작하기

Spring REST Docs는 아래의 요구사항을 가집니다.

- Java 8
- Spring Framework 5(5.0.2 or later)

### 빌드 설정

```kotlin
plugins {
  id("org.asciidoctor.jvm.convert") version "3.3.2" // 1.
}

val asciidoctorExt: Configuration by configurations.creating // 2.
val restDocVersion by extra("2.0.6.RELEASE")

dependencies {
  ...

  asciidoctorExt("org.springframework.restdocs:spring-restdocs-asciidoctor:${restDocVersion}") // 3.
  testImplementation("org.springframework.restdocs:spring-restdocs-webtestclient:${restDocVersion}") // 4.
}

val snippetsDir by extra(file("build/generated-snippets")) // 5.

tasks.test {
  outputs.dir(snippetsDir) // 6.
}

tasks.asciidoctor { // 7.
  inputs.dir(snippetsDir) // 8.
  configurations("asciidoctorExt") // 9.
  dependsOn(tasks.test) // 10.
}
```

1. asciidoctor 플러그인을 적용합니다. 현 시점(2022/3/8) 3.3.2 버전이 최신입니다.
2. asciidoctorExt 설정을 선언합니다. Kotlin DSL 에서는 Groovy DSL과 선언하는 방법이 다릅니다. [참고](https://docs.gradle.org/current/userguide/migrating_from_groovy_to_kotlin_dsl.html#custom_configurations_and_dependencies)
3. asciidoctorExt 설정으로 spring-restdocs-asciidoctor 의존성을 추가합니다. .adoc 형태의 스니펫을 만들어주는 역할을 합니다. 추후에 다룰 operation 매크로 사용을 가능하게 합니다.
4. spring-restdoc-webtestclient 의존성을 추가합니다. 저는 webflux를 쓰기 때문에 이 것을 사용했고 mvc, restassured 용도 따로 있었습니다.
5. 스니펫이 생성될 디렉토리를 정의합니다. Kotlin DSL에서는 위와 같이 정의합니다. [참고](https://docs.gradle.org/current/userguide/writing_build_scripts.html#sec:extra_properties)
6. test 작업의 output에 스니펫이 저장될 장소를 지정합니다.
7. asciidocker 작업을 설정합니다. 이 작업이 .adoc 파일을 .html로 변환하는 과정도 해줍니다.
8. 스니펫 디렉토리를 input 디렉토리로 지정합니다.
9. asciidocterExt 설정을 추가합니다.
10. test 작업의 의존성을 추가해서 해당 작업 전에 test가 수행되도록 합니다.

### 패키징

이 부분이 레퍼런스 가이드에서 이해하기 어려웠던 부분인데, 스프링 부트의 정적자원 서빙을 이용하여 문서를 제공하는 방법입니다. [참고](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-spring-mvc-static-content)

스프링 부트는 기본적으로 /static, /public, /resources, /META_INF/resources 위치를 정적자원으로 서빙합니다. 즉 서버가 localhost:8080에 떠있고 /static/docs/index.html 파일이 존재한다면 [localhost:8080/docs/index.html](http://localhost:8080/docs/index.html) 로 해당 파일을 접근할 수 있습니다.

```kotlin
tasks.bootJar {
  dependsOn(tasks.asciidoctor)
  from(tasks.asciidoctor.get().outputDir) {
    into("static/docs")
  }
}
```

asciidoctor 작업으로 생성된  html 파일을 static/docs 로 복사해주면 위에 언급한 주소에서 접근 가능해집니다.

### 문서 스니펫 작성

저는 JUnit5 로 테스트를 작성했습니다. 참고로 Webflux에서 컨트롤러를 테스트할 때는 @WebFluxTest 어노테이션으로 WebTestClient를 주입받을 수 있습니다. 하지만 AutoConfig를 사용하기 때문에 SpringRESTDocs의 설정을 추가할 수 없습니다. 그래서 다른 방법으로 테스트를 수행했습니다.

먼저 setup 메소드를 통해 WebTestClient 설정을 해줍니다.

```kotlin
@ExtendWith(RestDocumentationExtension::class)
class HelloWorldControllerSpec {
  private lateinit var webTestClient: WebTestClient
  private val helloWorldService = mock(HelloWorldService::class.java) // 1.
  private val subject = HelloWorldController(helloWorldService) // 2.

  @BeforeEach
  fun setUp(restDocumentation: RestDocumentationContextProvider) { // 3.
    WebTestClient.bindToController(subject)  // 4.
      .configureClient()
      .filter(documentationConfiguration(restDocumentation)) // 5.
      .entityExchangeResultConsumer(document("{class-name}/{method-name}")) // 6.
      .build()
  }  
}
```

1. HelloWorldController는 @Service 어노테이션으로 HelloWorldService를 주입받습니다. 하지만 우리는 단위테스트를 작성하는 것이 목적이므로 서비스를 모킹해줍니다.
2. 테스트에 사용할 컨트롤러를 생성합니다.
3. RestDocumentationExtension을 확장했기 때문에 setUp메서드에 restDocumentation 을 주입받을 수 있습니다.
4. WebTestClient.bindToController 스태틱 메서드를 통해 해당 컨트롤러만 테스트할 수 있도록 설정합니다.
5. 이 부분이 documentation을 가능하게 해줍니다.
6. 레퍼런스 가이드에서는 뒷 부분에 나오는 내용이지만 위와 같이 설정해주면 {outDir}/{클래스명}/{메소드명} 의 위치에 .adoc 파일들을 생성해줍니다.

### RESTFul 서비스 호출

이제 테스트를 작성할 차례입니다.

```kotlin
@Test
fun postHelloWorld() {
  `when`(helloWorldService.createHelloWorld(testHelloWorld))
      .thenReturn(Mono.just(testHelloWorld))

  webTestClient.post().uri("/v1/hello-world")
    .contentType(MediaType.APPLICATION_JSON)
    .accept(MediaType.APPLICATION_JSON)
    .bodyValue(testHelloWorld)
    .exchange()
    .expectStatus().isCreated
    .expectBody<HelloWorld>().isEqualTo(testHelloWorld)
//  .consumeWith(document("index") // 1.
}
```

여느 테스트와 다르지 않습니다. 다만, 1.에서 주석처리한 이유는 위에서 해당 설정을 class-name/method-name으로 해주었기 때문입니다. 문서 스니펫 작성 부분에서 6. 부분을 없애고 여기의 1.을 주석 해제하면 build/generated-snippets/index 위치에 .adoc 파일들이 생성됩니다.

### 스니펫 사용하기

이제 작성된 스니펫을 사용할 차례입니다. 스니펫을 사용하기 위해서는 Gradle의 경우 src/docs/asciidoc 의 위치에 .adoc 파일을 생성해야 합니다. 파일명은 마음대로 지어도 되고 지은 이름으로 .html 파일이 생성됩니다. 저는 편의상 index.adoc으로 짓겠습니다.

```kotlin
// src/main/asciidoc/index.adoc
= HelloWorld API Document
:doctype: book
:source-highlighter: highlightjs
:toc: left
:toclevels: 2
:sectlinks:

[[HelloWorld]]
== POST HelloWorld
operation::hello-world-controller-spec/post-hello-world[snippets='curl-request,http-request,http-response,request-body,response-body']
```

레퍼런스 가이드에는 asciidoctor 작업으로 생성된 html은 build/asciidoc/html5/ 위치에 생성된다고 되어있으나 제가 해보았을땐 html5 밑이 아닌 build/asciidoc 밑에 바로 생성되었습니다.

include 매크로를 통해 하나씩 가져와도 상관없지만 operation 매크로를 통해 원하는 것 만 더 손쉽게 가져올 수 있습니다.

### 문서 생성

이제 모든 준비가 끝났으니 문서를 생성할 차례입니다.

기본적으로 의존성을 엮어 놓았기 때문에 bootJar 작업으로 문서를 생성할 수 있으나, 각 작업이 하는 일을 나눠보겠습니다.

```bash
./gradlew clean test
```

위의 명령어로 테스트를 수행하면,

build/generated-snippets/hello-world-controller-spec/post-hello-world 의 위치에 아래의 파일들이 생성됩니다.

curl-request.adoc
http-request.adoc
http-response.adoc
httpie-request.adoc
request-body.adoc
response-body.adoc

이제 .adoc 파일을 .html 파일로 변경할 차례입니다.

```bash
./gradlew asciidoctor
```

asciidoctor 작업을 실행하면 test가 수행되는 것을 알 수 있습니다. 왜냐하면 의존성을 엮어 놓았기 때문입니다. 여기서는 각 단계에서 생성되는 산출물을 파악하기 위해 굳이 나눠서 실행했습니다.

build/docs/asciidoc 의 위치에 index.html 파일이 생성되었습니다.

이제 jar 파일을 만들어보겠습니다.

```bash
./gradlew bootJar
```

이 명령어를 수행하면, test, asciidoctor 작업이 다시 수행되는 것을 알 수 있습니다. CI 단계에서는 bootJar만 수행해도 충분할 것입니다. 이제 앱을 실행하고 문서가 잘 나오는지 확인합니다.

```bash
java -jar ./build/lib/xxx.jar
```

이제 [localhost:8080/docs/index.html](http://localhost:8080/docs/index.html) 로 접근하여 문서가 잘 나오는지 확인합니다.

그 외에 문서를 더 풍부하게 만드는 방법들이 레퍼런스 가이드에 소개되어 있습니다. [참고](https://docs.spring.io/spring-restdocs/docs/current/reference/html5/#documenting-your-api)

전 찍먹이기 때문에 여기까지 하겠습니다.

읽어주셔서 감사합니다.