# Java로 gRPC 빌드하기

## 작업 환경
- Mac OS
- Gradle 6.3
- IDE Jetbrain IntelliJ

## 프로토 파일 작성하기
기본적으로 protobuf-gradle-plugin은 src/$sourceSetName/proto  
위치에 있는 *.proto 파일을 컴파일 대상으로 하기 때문에,
$projectRoot/src/main/proto 위치에 $name.proto 파일을 작성했습니다.
(proto 파일의 위치를 변경할 수도 있습니다. https://github.com/google/protobuf-gradle-plugin 참고)
```proto
syntax = "proto3";

package post;

option java_package = "io.zillaci.grpc";
option java_outer_classname = "BlogProto";

message PostRequest {
    string title = 1;
    string body = 2;
    Author author = 3;
}

message Author {
    string name = 1;
    string email = 2;
}

message PostResponse {
    int32 post_id = 1;
}

service Blog {
    rpc AddPost (PostRequest) returns (PostResponse) {}
}
```

## Gradle 빌드하기
이제 proto 파일을 Java 프로젝트에서 사용할 수 있도록 컴파일합니다.  
gradle 로 편하게 빌드하기 위해 plugin 을 적용했습니다.
protobuf-gradle-plugin 은 build task 에 자동으로 proto 컴파일 관련 task 들을 추가해줍니다.
수동으로 task 를 만들 수도 있으나 편의상 plugin 을 사용했습니다.

### protobuf-gradle-plugin
해당 플러그인은 반드시 java 또는 android 플러그인과 함께 쓰여야 합니다.
또한, 빌드를 해보면 컴파일에 필요한 java 패키지가 없다는 에러가 발생하는데, protobuf-java 패키지를 추가해줍니다.
```groovy
plugin {
  id "java" // java 또는 android 플러그인이 먼저 선언되어야 합니다.
  id "com.google.protobuf" version "0.8.12"
  ...
}
...
dependencies {
  ...
  implementation "com.google.protobuf:protobuf-java:3.11.4"
  ...
}
```
이제 빌드를 하면 build 디렉토리 밑에 io.zillaci.grpc.BlogProto.class 파일이 생성된 것을 확인할 수 있습니다.
```bash
./gradlew build
ls build/classes/java/main/io/zillaci/grpc
ls build/libs # jar 파일은 이곳에 생성됩니다.
```

빌드 도중 protoc가 설치되어 있지 않으면 아래와 같은 에러가 발생할 수 있습니다.  
http://google.github.io/proto-lens/installing-protoc.html 를 참고하여 OS에 맞는 protobuf를 설치하면 됩니다.
```bash
* What went wrong:
Execution failed for task ':generateProto'.
> java.io.IOException: Cannot run program "protoc": error=2, No such file or directory
```

### References
- https://developers.google.com/protocol-buffers/docs/overview
- https://github.com/google/protobuf-gradle-plugin
- http://google.github.io/proto-lens/installing-protoc.html
- https://www.baeldung.com/grpc-introduction