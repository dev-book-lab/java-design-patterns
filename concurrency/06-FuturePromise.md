# Future/Promise Pattern (퓨처/프로미스 패턴)

> **"비동기 작업의 결과를 미래에 받을 수 있게 하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [CompletableFuture 완전 가이드](#6-completablefuture-완전-가이드)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 비동기 작업의 결과 받기 어려움
public class DataService {
    public void fetchData() {
        new Thread(() -> {
            String result = downloadData();
            
            // 😱 result를 어떻게 반환?
            // return? → void라서 불가능!
            // 필드에 저장? → Thread-Safe 문제!
        }).start();
    }
}

// 문제 2: Callback Hell
public class ApiClient {
    public void fetchUserData(String userId) {
        getUserProfile(userId, profile -> {
            // 프로필 받고
            getOrders(profile.getId(), orders -> {
                // 주문 받고
                getOrderDetails(orders.get(0).getId(), details -> {
                    // 상세 정보 받고
                    processDetails(details, result -> {
                        // 😱 중첩이 너무 깊음! (Callback Hell)
                        System.out.println(result);
                    });
                });
            });
        });
    }
}

// 문제 3: 에러 처리 복잡
ExecutorService executor = Executors.newFixedThreadPool(10);

executor.submit(() -> {
    try {
        return fetchData();
    } catch (Exception e) {
        // 😱 예외를 어디로?
        e.printStackTrace();
        return null;
    }
});

// 예외가 조용히 사라짐!

// 문제 4: 여러 비동기 작업 조합
public class ParallelFetcher {
    public void fetchAll() {
        // 😱 3개 API를 병렬로 호출하고 모두 완료되면 처리?
        
        Thread t1 = new Thread(() -> {
            String user = fetchUser();
            // 어떻게 합칠까?
        });
        
        Thread t2 = new Thread(() -> {
            String orders = fetchOrders();
        });
        
        Thread t3 = new Thread(() -> {
            String products = fetchProducts();
        });
        
        t1.start();
        t2.start();
        t3.start();
        
        // 모두 완료 대기?
        t1.join();
        t2.join();
        t3.join();
        
        // 복잡하고 에러 발생 쉬움!
    }
}

// 문제 5: 타임아웃 처리
public class TimeoutExample {
    public void fetchWithTimeout() {
        // 😱 3초 안에 완료 안 되면 취소?
        
        Thread thread = new Thread(() -> {
            fetchData();  // 느린 작업
        });
        
        thread.start();
        thread.join(3000);  // 3초 대기
        
        if (thread.isAlive()) {
            thread.interrupt();  // 취소 시도
            // 실제로 취소되는지 보장 안 됨!
        }
    }
}

// 문제 6: 순차/병렬 조합
public class ChainExample {
    public void processChain() {
        // 😱 A 완료 → B 실행 → C와 D 병렬 → 모두 완료 후 E
        
        // 이걸 어떻게?
    }
}
```

### ⚡ 핵심 문제

1. **결과 반환**: 비동기 결과 받기 어려움
2. **Callback Hell**: 중첩된 콜백
3. **에러 처리**: 예외 전파 복잡
4. **조합**: 여러 작업 조합 어려움
5. **타임아웃**: 시간 제한 구현 복잡

---

## 2. 패턴 정의

### 📖 정의

> **비동기 작업의 결과를 나타내는 객체로, 작업이 완료되면 결과를 제공하거나 실패 시 예외를 제공하는 패턴**

### 🎯 목적

- **비동기 결과**: 미래 값 표현
- **체이닝**: 작업 연결
- **에러 처리**: 예외 전파
- **조합**: 여러 작업 조합

### 💡 핵심 아이디어

```java
// Before: Callback
fetchData(result -> {
    process(result, processed -> {
        save(processed, saved -> {
            // 😱 중첩
        });
    });
});

// After: Future (체이닝)
CompletableFuture.supplyAsync(() -> fetchData())
    .thenApply(this::process)
    .thenApply(this::save)
    .thenAccept(System.out::println);
// 😊 읽기 쉬움!
```

---

## 3. 구조와 구성요소

### 📊 Future 구조

```
┌─────────────────────────┐
│    CompletableFuture    │
│                         │
│  - 상태: Pending         │
│  - 결과: null            │
└─────────────────────────┘
         │
         ▼ 비동기 실행
┌─────────────────────────┐
│  작업 실행 중...           │
└─────────────────────────┘
         │
         ▼ 완료
┌─────────────────────────┐
│  - 상태: Completed       │
│  - 결과: "Success"       │
└─────────────────────────┘
```

---

## 4. 구현 방법

### 완전한 구현: API 클라이언트 ⭐⭐⭐

```java
/**
 * ============================================
 * BASIC FUTURE
 * ============================================
 */

/**
 * 기본 Future 예제
 */
public class BasicFutureExample {
    private final ExecutorService executor = Executors.newFixedThreadPool(10);
    
    /**
     * Future 사용
     */
    public void basicFutureExample() throws Exception {
        System.out.println("\n=== 기본 Future ===");
        
        // 비동기 작업 제출
        Future<String> future = executor.submit(() -> {
            System.out.println("⚙️ 작업 실행 중...");
            Thread.sleep(2000);
            return "결과";
        });
        
        System.out.println("✅ 작업 제출 완료 (즉시 반환)");
        
        // 다른 작업...
        System.out.println("💼 다른 작업 수행 중...");
        Thread.sleep(1000);
        
        // 결과 대기
        System.out.println("⏳ 결과 대기 중...");
        String result = future.get();  // Blocking
        
        System.out.println("📦 결과: " + result);
    }
}

/**
 * ============================================
 * COMPLETABLEFUTURE BASIC
 * ============================================
 */

/**
 * User 클래스
 */
class User {
    private final Long id;
    private final String name;
    
    public User(Long id, String name) {
        this.id = id;
        this.name = name;
    }
    
    public Long getId() { return id; }
    public String getName() { return name; }
    
    @Override
    public String toString() {
        return "User{id=" + id + ", name='" + name + "'}";
    }
}

/**
 * CompletableFuture 예제
 */
public class CompletableFutureExamples {
    
    /**
     * 1. supplyAsync (결과 반환)
     */
    public void supplyAsyncExample() throws Exception {
        System.out.println("\n=== supplyAsync ===");
        
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            System.out.println("⚙️ 비동기 작업 실행");
            sleep(1000);
            return "Hello CompletableFuture";
        });
        
        System.out.println("✅ Future 생성 (비동기 실행 중)");
        
        String result = future.get();
        System.out.println("📦 결과: " + result);
    }
    
    /**
     * 2. thenApply (변환)
     */
    public void thenApplyExample() throws Exception {
        System.out.println("\n=== thenApply (체이닝) ===");
        
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            System.out.println("⚙️ Step 1: 데이터 가져오기");
            sleep(500);
            return "DATA";
        }).thenApply(data -> {
            System.out.println("⚙️ Step 2: 데이터 변환 (" + data + ")");
            sleep(500);
            return data.toLowerCase();
        }).thenApply(data -> {
            System.out.println("⚙️ Step 3: 접두사 추가 (" + data + ")");
            sleep(500);
            return "processed_" + data;
        });
        
        String result = future.get();
        System.out.println("📦 최종 결과: " + result);
    }
    
    /**
     * 3. thenCompose (Flat Map)
     */
    public void thenComposeExample() throws Exception {
        System.out.println("\n=== thenCompose ===");
        
        CompletableFuture<User> future = CompletableFuture.supplyAsync(() -> {
            System.out.println("⚙️ 사용자 ID 조회");
            sleep(500);
            return 1L;
        }).thenCompose(userId -> {
            System.out.println("⚙️ 사용자 정보 조회: " + userId);
            return CompletableFuture.supplyAsync(() -> {
                sleep(500);
                return new User(userId, "Alice");
            });
        });
        
        User user = future.get();
        System.out.println("📦 사용자: " + user);
    }
    
    /**
     * 4. thenCombine (2개 Future 조합)
     */
    public void thenCombineExample() throws Exception {
        System.out.println("\n=== thenCombine ===");
        
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
            System.out.println("⚙️ API 1 호출");
            sleep(1000);
            return "Data1";
        });
        
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
            System.out.println("⚙️ API 2 호출");
            sleep(1000);
            return "Data2";
        });
        
        CompletableFuture<String> combined = future1.thenCombine(future2, (data1, data2) -> {
            System.out.println("⚙️ 결과 합치기");
            return data1 + " + " + data2;
        });
        
        String result = combined.get();
        System.out.println("📦 합친 결과: " + result);
    }
    
    /**
     * 5. allOf (여러 Future 모두 대기)
     */
    public void allOfExample() throws Exception {
        System.out.println("\n=== allOf ===");
        
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
            System.out.println("⚙️ Task 1");
            sleep(1000);
            return "Result1";
        });
        
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
            System.out.println("⚙️ Task 2");
            sleep(1500);
            return "Result2";
        });
        
        CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
            System.out.println("⚙️ Task 3");
            sleep(800);
            return "Result3";
        });
        
        CompletableFuture<Void> allOf = CompletableFuture.allOf(future1, future2, future3);
        
        System.out.println("⏳ 모든 작업 완료 대기...");
        allOf.get();
        
        System.out.println("✅ 모든 작업 완료!");
        System.out.println("   Result1: " + future1.get());
        System.out.println("   Result2: " + future2.get());
        System.out.println("   Result3: " + future3.get());
    }
    
    /**
     * 6. anyOf (가장 빠른 것 하나)
     */
    public void anyOfExample() throws Exception {
        System.out.println("\n=== anyOf ===");
        
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
            System.out.println("⚙️ Slow API");
            sleep(3000);
            return "Slow Result";
        });
        
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
            System.out.println("⚙️ Fast API");
            sleep(500);
            return "Fast Result";
        });
        
        CompletableFuture<Object> anyOf = CompletableFuture.anyOf(future1, future2);
        
        System.out.println("⏳ 가장 빠른 결과 대기...");
        Object result = anyOf.get();
        
        System.out.println("📦 첫 번째 결과: " + result);
    }
    
    /**
     * 7. exceptionally (에러 처리)
     */
    public void exceptionallyExample() throws Exception {
        System.out.println("\n=== exceptionally ===");
        
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            System.out.println("⚙️ 작업 실행 (에러 발생 예정)");
            sleep(500);
            throw new RuntimeException("오류 발생!");
        }).exceptionally(ex -> {
            System.err.println("❌ 에러 처리: " + ex.getMessage());
            return "기본값";
        });
        
        String result = future.get();
        System.out.println("📦 결과: " + result);
    }
    
    /**
     * 8. handle (성공/실패 모두 처리)
     */
    public void handleExample() throws Exception {
        System.out.println("\n=== handle ===");
        
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            System.out.println("⚙️ 작업 실행");
            sleep(500);
            if (Math.random() > 0.5) {
                throw new RuntimeException("랜덤 에러");
            }
            return "Success";
        }).handle((result, ex) -> {
            if (ex != null) {
                System.err.println("❌ 에러: " + ex.getMessage());
                return "Error Handled";
            } else {
                System.out.println("✅ 성공: " + result);
                return result;
            }
        });
        
        String result = future.get();
        System.out.println("📦 최종 결과: " + result);
    }
    
    private void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

/**
 * ============================================
 * COMPLETE EXAMPLE: API CLIENT
 * ============================================
 */

/**
 * API 클라이언트
 */
public class ApiClient {
    
    /**
     * 사용자 조회
     */
    public CompletableFuture<User> getUser(Long userId) {
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("🔍 사용자 조회: " + userId);
            sleep(1000);
            return new User(userId, "User-" + userId);
        });
    }
    
    /**
     * 주문 조회
     */
    public CompletableFuture<List<String>> getOrders(Long userId) {
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("📦 주문 조회: " + userId);
            sleep(1500);
            return List.of("Order1", "Order2", "Order3");
        });
    }
    
    /**
     * 추천 상품 조회
     */
    public CompletableFuture<List<String>> getRecommendations(Long userId) {
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("⭐ 추천 상품 조회: " + userId);
            sleep(800);
            return List.of("Product1", "Product2");
        });
    }
    
    /**
     * 복합 조회 (병렬)
     */
    public CompletableFuture<DashboardData> getDashboard(Long userId) {
        System.out.println("\n📊 대시보드 조회 시작");
        
        // 3개 API 병렬 호출
        CompletableFuture<User> userFuture = getUser(userId);
        CompletableFuture<List<String>> ordersFuture = getOrders(userId);
        CompletableFuture<List<String>> recommendationsFuture = getRecommendations(userId);
        
        // 모두 완료 후 조합
        return CompletableFuture.allOf(userFuture, ordersFuture, recommendationsFuture)
            .thenApply(v -> {
                System.out.println("✅ 모든 API 완료, 데이터 조합 중");
                
                User user = userFuture.join();
                List<String> orders = ordersFuture.join();
                List<String> recommendations = recommendationsFuture.join();
                
                return new DashboardData(user, orders, recommendations);
            });
    }
    
    private void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

/**
 * 대시보드 데이터
 */
record DashboardData(User user, List<String> orders, List<String> recommendations) {
    @Override
    public String toString() {
        return "Dashboard{\n" +
               "  user=" + user + ",\n" +
               "  orders=" + orders + ",\n" +
               "  recommendations=" + recommendations + "\n" +
               "}";
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class FuturePromiseDemo {
    public static void main(String[] args) throws Exception {
        System.out.println("=== Future/Promise Pattern 예제 ===");
        
        CompletableFutureExamples examples = new CompletableFutureExamples();
        
        // 1. 기본
        examples.supplyAsyncExample();
        
        // 2. 체이닝
        examples.thenApplyExample();
        
        // 3. Compose
        examples.thenComposeExample();
        
        // 4. Combine
        examples.thenCombineExample();
        
        // 5. AllOf
        examples.allOfExample();
        
        // 6. AnyOf
        examples.anyOfExample();
        
        // 7. 에러 처리
        examples.exceptionallyExample();
        examples.handleExample();
        
        // 8. 복합 예제
        System.out.println("\n" + "=".repeat(60));
        System.out.println("복합 API 호출");
        System.out.println("=".repeat(60));
        
        ApiClient apiClient = new ApiClient();
        
        long start = System.currentTimeMillis();
        
        CompletableFuture<DashboardData> dashboardFuture = apiClient.getDashboard(1L);
        
        DashboardData dashboard = dashboardFuture.get();
        
        long elapsed = System.currentTimeMillis() - start;
        
        System.out.println("\n📊 대시보드 데이터:");
        System.out.println(dashboard);
        System.out.println("\n⏱️ 소요 시간: " + elapsed + "ms (병렬 처리!)");
        
        System.out.println("\n✅ 모든 예제 완료!");
    }
}
```

**실행 결과:**
```
=== Future/Promise Pattern 예제 ===

=== supplyAsync ===
✅ Future 생성 (비동기 실행 중)
⚙️ 비동기 작업 실행
📦 결과: Hello CompletableFuture

=== thenApply (체이닝) ===
⚙️ Step 1: 데이터 가져오기
⚙️ Step 2: 데이터 변환 (DATA)
⚙️ Step 3: 접두사 추가 (data)
📦 최종 결과: processed_data

=== thenCompose ===
⚙️ 사용자 ID 조회
⚙️ 사용자 정보 조회: 1
📦 사용자: User{id=1, name='Alice'}

=== thenCombine ===
⚙️ API 1 호출
⚙️ API 2 호출
⚙️ 결과 합치기
📦 합친 결과: Data1 + Data2

=== allOf ===
⚙️ Task 1
⚙️ Task 2
⚙️ Task 3
⏳ 모든 작업 완료 대기...
✅ 모든 작업 완료!
   Result1: Result1
   Result2: Result2
   Result3: Result3

============================================================
복합 API 호출
============================================================

📊 대시보드 조회 시작
🔍 사용자 조회: 1
📦 주문 조회: 1
⭐ 추천 상품 조회: 1
✅ 모든 API 완료, 데이터 조합 중

📊 대시보드 데이터:
Dashboard{
  user=User{id=1, name='User-1'},
  orders=[Order1, Order2, Order3],
  recommendations=[Product1, Product2]
}

⏱️ 소요 시간: 1503ms (병렬 처리!)

✅ 모든 예제 완료!
```

---

## 5. 실전 예제

### 예제 1: 파일 다운로드 ⭐⭐⭐

```java
public CompletableFuture<byte[]> downloadFile(String url) {
    return CompletableFuture.supplyAsync(() -> {
        // 다운로드
        return downloadBytes(url);
    }).orTimeout(10, TimeUnit.SECONDS)  // 타임아웃
      .exceptionally(ex -> {
          // 에러 시 재시도
          return retryDownload(url);
      });
}
```

---

## 6. CompletableFuture 완전 가이드

### 📊 주요 메서드

| 메서드 | 설명 |
|--------|------|
| `supplyAsync` | 비동기 실행 (결과 반환) |
| `thenApply` | 변환 |
| `thenCompose` | FlatMap |
| `thenCombine` | 2개 조합 |
| `allOf` | 모두 대기 |
| `anyOf` | 하나 대기 |
| `exceptionally` | 에러 처리 |

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **비동기 결과** | 미래 값 표현 |
| **체이닝** | 작업 연결 |
| **조합** | 여러 작업 조합 |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **복잡도** | 학습 필요 |
| **디버깅** | 어려움 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: Blocking get()

```java
// 잘못된 예
CompletableFuture<String> future = fetchDataAsync();
String result = future.get();  // ❌ Blocking!
process(result);

// 올바른 예
fetchDataAsync()
    .thenAccept(this::process);  // ✅ 비동기
```

---

## 9. 심화 주제

### 🎯 Reactive Streams

```java
// Project Reactor
Mono<String> mono = Mono.fromFuture(
    CompletableFuture.supplyAsync(() -> "Data")
);
```

---

## 10. 핵심 정리

### 📌 체크리스트

```
✅ CompletableFuture 사용
✅ 체이닝으로 연결
✅ exceptionally로 에러 처리
✅ allOf/anyOf로 조합
✅ get() 대신 thenAccept
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Active Object](./05-ActiveObject.md)**

</div>
