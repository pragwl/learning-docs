---

# 📘 gRPC in Java with Spring Boot & Gradle — Full Notes

---

## 🔹 1. What is gRPC?

* gRPC = Google Remote Procedure Call
* Uses **Protocol Buffers** (Protobuf) for message serialization
* Based on **HTTP/2** → fast, multiplexed, streaming
* Supports multiple communication types (Unary, Streaming)
* Language-neutral, great for microservices

---

## 🔹 2. Why gRPC over REST?

| Feature          | REST    | gRPC          |
| ---------------- | ------- | ------------- |
| Data Format      | JSON    | Protobuf      |
| Speed            | Slower  | Faster        |
| Contract         | OpenAPI | `.proto` file |
| Streaming        | Limited | Built-in      |
| Language Support | Broad   | Broad         |

---

## 🔹 3. Project Setup: Gradle + Spring Boot + gRPC

### 📁 Folder Structure

```
src/
 └── main/
     ├── java/              ← Spring Boot app
     └── proto/             ← .proto files
build.gradle
```

---

### 🔧 `build.gradle` (Groovy DSL)

```groovy
plugins {
  id 'java'
  id 'org.springframework.boot' version '3.2.0'
  id 'com.google.protobuf' version '0.9.4'
}

group = 'com.example'
version = '1.0'
sourceCompatibility = '17'

repositories {
  mavenCentral()
}

dependencies {
  implementation 'org.springframework.boot:spring-boot-starter'
  implementation 'net.devh:grpc-server-spring-boot-starter:2.14.0.RELEASE'
  implementation 'net.devh:grpc-client-spring-boot-starter:2.14.0.RELEASE'

  compileOnly 'org.projectlombok:lombok:1.18.28'
  annotationProcessor 'org.projectlombok:lombok:1.18.28'
}

protobuf {
  protoc {
    artifact = 'com.google.protobuf:protoc:3.25.3'
  }
  plugins {
    grpc {
      artifact = 'io.grpc:protoc-gen-grpc-java:1.63.0'
    }
  }
  generateProtoTasks {
    all().each { task ->
      task.plugins {
        grpc {}
      }
    }
  }
}

```

---

## 🔹 4. Define Proto File

### 📄 `src/main/proto/greeter.proto`

```proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "com.example.grpc";
option java_outer_classname = "GreeterProto";

service GreeterService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string name = 1;
}

message HelloResponse {
  string message = 1;
}
```

---

## 🔹 5. Generate Code (Automatically on build)

* Gradle plugin will compile `.proto` into:

  * `GreeterServiceGrpc.java`
  * `HelloRequest.java`
  * `HelloResponse.java`

---

## 🔹 6. Create gRPC Server (Spring Boot)

### 📄 `GreeterServiceImpl.java`

```java
@Service
public class GreeterServiceImpl extends GreeterServiceGrpc.GreeterServiceImplBase {

  @Override
  public void sayHello(HelloRequest request, StreamObserver<HelloResponse> responseObserver) {
    String message = "Hello, " + request.getName();
    HelloResponse response = HelloResponse.newBuilder()
        .setMessage(message)
        .build();
    responseObserver.onNext(response);
    responseObserver.onCompleted();
  }
}
```

---

## 🔹 7. Application Class

```java
@SpringBootApplication
public class GrpcDemoApplication {
  public static void main(String[] args) {
    SpringApplication.run(GrpcDemoApplication.class, args);
  }
}
```

---

## 🔹 8. Client Side (Java Spring Boot)

### 📄 `GreeterClient.java`

```java
@Component
public class GreeterClient {

  @GrpcClient("local-grpc-server")
  private GreeterServiceGrpc.GreeterServiceBlockingStub stub;

  public void greet(String name) {
    HelloRequest request = HelloRequest.newBuilder().setName(name).build();
    HelloResponse response = stub.sayHello(request);
    System.out.println("Response: " + response.getMessage());
  }
}
```

---

## 🔹 9. application.yml (Optional)

```yaml
grpc:
  client:
    local-grpc-server:
      address: static://localhost:9090
      negotiationType: plaintext
```

---

## 🔹 10. gRPC Communication Types (in Java)

| Type                    | Example               |
| ----------------------- | --------------------- |
| Unary                   | `sayHello()`          |
| Server Streaming        | Real-time price feed  |
| Client Streaming        | Upload logs in chunks |
| Bidirectional Streaming | Chat / live games     |

Implement these by overriding the corresponding methods:

```java
public void chat(StreamObserver<ChatRequest> request, StreamObserver<ChatResponse> response) { ... }
```

---

## 🔹 11. Advanced: Security (API Key or TLS)

Use Spring Boot interceptors or `NettyServerBuilder` to add TLS and metadata authentication.

---

## 🧪 12. Test gRPC Locally

Use Postman (with gRPC support) or CLI tools like `grpcurl`:

```bash
grpcurl -plaintext localhost:9090 list
grpcurl -plaintext -d '{"name":"Priyank"}' localhost:9090 com.example.grpc.GreeterService/SayHello
```

---

## ✅ Summary: gRPC with Spring Boot & Gradle

| Area             | What You Do                         |
| ---------------- | ----------------------------------- |
| Define proto     | In `src/main/proto/*.proto`         |
| Generate code    | Gradle plugin `protobuf` handles it |
| Write service    | Extend `*ImplBase` class            |
| Start server     | `@Service` + Spring Boot runner     |
| Call from client | Use `@GrpcClient` in Spring         |
| Secure it        | TLS, metadata interceptors          |

---
