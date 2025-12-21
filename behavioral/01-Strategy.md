# Strategy Pattern (전략 패턴)

> **"알고리즘을 캡슐화하여 교체 가능하게 만들자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [장단점](#6-장단점)
7. [안티패턴](#7-안티패턴)
8. [핵심 정리](#8-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: if-else 지옥
public class PaymentService {
    public void processPayment(String method, double amount) {
        if (method.equals("CREDIT_CARD")) {
            // 신용카드 결제 로직 (50줄)
            System.out.println("신용카드 결제...");
            validateCard();
            connectToCardGateway();
            processCardPayment(amount);
            
        } else if (method.equals("PAYPAL")) {
            // PayPal 결제 로직 (50줄)
            System.out.println("PayPal 결제...");
            validatePayPalAccount();
            connectToPayPal();
            processPayPalPayment(amount);
            
        } else if (method.equals("BANK_TRANSFER")) {
            // 계좌이체 로직 (50줄)
            System.out.println("계좌이체...");
            validateBankAccount();
            connectToBank();
            processBankTransfer(amount);
        }
        
        // 새 결제 수단 추가 시 이 메서드 수정! (OCP 위반)
    }
}

// 문제 2: 알고리즘 변경이 어려움
public class DataCompressor {
    private String compressionType = "ZIP";
    
    public byte[] compress(byte[] data) {
        if (compressionType.equals("ZIP")) {
            return zipCompress(data);
        } else if (compressionType.equals("GZIP")) {
            return gzipCompress(data);
        } else if (compressionType.equals("RAR")) {
            return rarCompress(data);
        }
        return data;
    }
    
    // 압축 알고리즘 변경하려면 클래스 수정!
}

// 문제 3: 테스트가 어려움
public class NavigationService {
    public Route findRoute(Location from, Location to, String mode) {
        if (mode.equals("CAR")) {
            // 자동차 경로 계산 (복잡한 로직)
            return calculateCarRoute(from, to);
        } else if (mode.equals("WALK")) {
            // 도보 경로 계산 (복잡한 로직)
            return calculateWalkRoute(from, to);
        } else if (mode.equals("BIKE")) {
            // 자전거 경로 계산 (복잡한 로직)
            return calculateBikeRoute(from, to);
        }
        return null;
    }
    
    // 각 알고리즘을 개별적으로 테스트하기 어려움!
}

// 문제 4: 중복 코드
public class SortingService {
    public void sort(int[] arr, String algorithm) {
        if (algorithm.equals("BUBBLE")) {
            // 버블 정렬
            for (int i = 0; i < arr.length; i++) {
                for (int j = 0; j < arr.length - 1; j++) {
                    if (arr[j] > arr[j + 1]) {
                        int temp = arr[j];
                        arr[j] = arr[j + 1];
                        arr[j + 1] = temp;
                    }
                }
            }
        } else if (algorithm.equals("QUICK")) {
            // 퀵 정렬 (복잡한 로직)
            quickSort(arr, 0, arr.length - 1);
        }
        // 알고리즘마다 조건문 안에 전체 구현!
    }
}
```

### ⚡ 핵심 문제

1. **OCP 위반**: 새 알고리즘 추가 시 기존 코드 수정
2. **낮은 응집도**: 여러 알고리즘이 한 클래스에
3. **테스트 어려움**: 알고리즘을 독립적으로 테스트 불가
4. **재사용 불가**: 알고리즘을 다른 곳에서 사용 어려움

---

## 2. 패턴 정의

### 📖 정의

> **알고리즘군을 정의하고 각각을 캡슐화하여 교환 가능하게 만드는 패턴. 알고리즘을 사용하는 클라이언트와 독립적으로 알고리즘을 변경할 수 있다.**

### 🎯 목적

- **알고리즘 캡슐화**: 각 알고리즘을 독립 클래스로
- **교체 가능**: 런타임에 알고리즘 변경
- **OCP 준수**: 새 알고리즘 추가 시 기존 코드 불변
- **재사용성**: 알고리즘을 여러 곳에서 사용

### 💡 핵심 아이디어

```java
// Before: if-else로 알고리즘 선택
if (type.equals("A")) {
    // 알고리즘 A
} else if (type.equals("B")) {
    // 알고리즘 B
}

// After: Strategy로 알고리즘 교체
Strategy strategy = new StrategyA();
strategy.execute();

strategy = new StrategyB(); // 런타임 교체!
strategy.execute();
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌─────────────────┐
│    Context      │  ← 전략 사용자
├─────────────────┤
│ - strategy      │───┐
│ + setStrategy() │   │ has-a
│ + execute()     │   │
└─────────────────┘   │
                      │
                      ▼
              ┌───────────────┐
              │   Strategy    │  ← 전략 인터페이스
              ├───────────────┤
              │ + algorithm() │
              └───────────────┘
                      △
                      │ implements
         ┌────────────┼────────────┐
         │            │            │
┌────────────┐ ┌────────────┐ ┌────────────┐
│StrategyA   │ │StrategyB   │ │StrategyC   │
├────────────┤ ├────────────┤ ├────────────┤
│algorithm() │ │algorithm() │ │algorithm() │
└────────────┘ └────────────┘ └────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Strategy** | 알고리즘 인터페이스 | `PaymentStrategy` |
| **ConcreteStrategy** | 구체적 알고리즘 | `CreditCardStrategy` |
| **Context** | 전략 사용 | `PaymentService` |

---

## 4. 구현 방법

### 기본 구현: 결제 시스템 ⭐⭐⭐

```java
/**
 * Strategy: 결제 전략 인터페이스
 */
public interface PaymentStrategy {
    void pay(double amount);
    String getPaymentMethod();
}

/**
 * ConcreteStrategy 1: 신용카드 결제
 */
public class CreditCardStrategy implements PaymentStrategy {
    private String cardNumber;
    private String cvv;
    private String expiryDate;
    
    public CreditCardStrategy(String cardNumber, String cvv, String expiryDate) {
        this.cardNumber = cardNumber;
        this.cvv = cvv;
        this.expiryDate = expiryDate;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("💳 신용카드 결제 처리 중...");
        System.out.println("   카드번호: " + maskCardNumber(cardNumber));
        System.out.println("   금액: $" + amount);
        System.out.println("   ✅ 결제 완료!");
    }
    
    @Override
    public String getPaymentMethod() {
        return "Credit Card";
    }
    
    private String maskCardNumber(String cardNumber) {
        return "****-****-****-" + cardNumber.substring(cardNumber.length() - 4);
    }
}

/**
 * ConcreteStrategy 2: PayPal 결제
 */
public class PayPalStrategy implements PaymentStrategy {
    private String email;
    private String password;
    
    public PayPalStrategy(String email, String password) {
        this.email = email;
        this.password = password;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("💰 PayPal 결제 처리 중...");
        System.out.println("   계정: " + email);
        System.out.println("   금액: $" + amount);
        System.out.println("   ✅ 결제 완료!");
    }
    
    @Override
    public String getPaymentMethod() {
        return "PayPal";
    }
}

/**
 * ConcreteStrategy 3: 계좌이체
 */
public class BankTransferStrategy implements PaymentStrategy {
    private String bankName;
    private String accountNumber;
    
    public BankTransferStrategy(String bankName, String accountNumber) {
        this.bankName = bankName;
        this.accountNumber = accountNumber;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("🏦 계좌이체 처리 중...");
        System.out.println("   은행: " + bankName);
        System.out.println("   계좌: " + accountNumber);
        System.out.println("   금액: $" + amount);
        System.out.println("   ✅ 이체 완료!");
    }
    
    @Override
    public String getPaymentMethod() {
        return "Bank Transfer";
    }
}

/**
 * Context: 쇼핑카트
 */
public class ShoppingCart {
    private List<Item> items;
    private PaymentStrategy paymentStrategy;
    
    public ShoppingCart() {
        this.items = new ArrayList<>();
    }
    
    public void addItem(Item item) {
        items.add(item);
        System.out.println("🛒 장바구니에 추가: " + item);
    }
    
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
        System.out.println("💡 결제 방법 선택: " + strategy.getPaymentMethod());
    }
    
    public void checkout() {
        double total = calculateTotal();
        
        System.out.println("\n=== 결제 진행 ===");
        System.out.println("총 금액: $" + total);
        
        if (paymentStrategy == null) {
            System.out.println("❌ 결제 방법을 선택해주세요!");
            return;
        }
        
        paymentStrategy.pay(total);
    }
    
    private double calculateTotal() {
        return items.stream()
                .mapToDouble(Item::getPrice)
                .sum();
    }
}

/**
 * 상품 클래스
 */
class Item {
    private String name;
    private double price;
    
    public Item(String name, double price) {
        this.name = name;
        this.price = price;
    }
    
    public double getPrice() {
        return price;
    }
    
    @Override
    public String toString() {
        return name + " ($" + price + ")";
    }
}

/**
 * 사용 예제
 */
public class StrategyExample {
    public static void main(String[] args) {
        // 쇼핑카트 생성
        ShoppingCart cart = new ShoppingCart();
        
        // 상품 추가
        System.out.println("=== 쇼핑 시작 ===");
        cart.addItem(new Item("노트북", 1200.00));
        cart.addItem(new Item("마우스", 25.00));
        cart.addItem(new Item("키보드", 75.00));
        
        // 신용카드로 결제
        System.out.println("\n--- 신용카드 결제 ---");
        cart.setPaymentStrategy(new CreditCardStrategy(
            "1234567890123456", "123", "12/25"
        ));
        cart.checkout();
        
        // PayPal로 결제 (전략 교체!)
        System.out.println("\n--- PayPal로 변경 ---");
        cart.setPaymentStrategy(new PayPalStrategy(
            "user@example.com", "password"
        ));
        cart.checkout();
        
        // 계좌이체로 결제
        System.out.println("\n--- 계좌이체로 변경 ---");
        cart.setPaymentStrategy(new BankTransferStrategy(
            "KB국민은행", "123-456-789"
        ));
        cart.checkout();
    }
}
```

**실행 결과:**
```
=== 쇼핑 시작 ===
🛒 장바구니에 추가: 노트북 ($1200.0)
🛒 장바구니에 추가: 마우스 ($25.0)
🛒 장바구니에 추가: 키보드 ($75.0)

--- 신용카드 결제 ---
💡 결제 방법 선택: Credit Card

=== 결제 진행 ===
총 금액: $1300.0
💳 신용카드 결제 처리 중...
   카드번호: ****-****-****-3456
   금액: $1300.0
   ✅ 결제 완료!

--- PayPal로 변경 ---
💡 결제 방법 선택: PayPal

=== 결제 진행 ===
총 금액: $1300.0
💰 PayPal 결제 처리 중...
   계정: user@example.com
   금액: $1300.0
   ✅ 결제 완료!

--- 계좌이체로 변경 ---
💡 결제 방법 선택: Bank Transfer

=== 결제 진행 ===
총 금액: $1300.0
🏦 계좌이체 처리 중...
   은행: KB국민은행
   계좌: 123-456-789
   금액: $1300.0
   ✅ 이체 완료!
```

---

## 5. 실전 예제

### 예제 1: 내비게이션 시스템 ⭐⭐⭐

```java
/**
 * Strategy: 경로 탐색 전략
 */
public interface RouteStrategy {
    void buildRoute(String from, String to);
    int estimateTime(String from, String to);
}

/**
 * ConcreteStrategy: 자동차 경로
 */
public class CarRouteStrategy implements RouteStrategy {
    @Override
    public void buildRoute(String from, String to) {
        System.out.println("🚗 자동차 경로 계산 중...");
        System.out.println("   출발: " + from);
        System.out.println("   도착: " + to);
        System.out.println("   경로: 고속도로 이용");
        System.out.println("   거리: 45 km");
    }
    
    @Override
    public int estimateTime(String from, String to) {
        return 30; // 30분
    }
}

/**
 * ConcreteStrategy: 도보 경로
 */
public class WalkRouteStrategy implements RouteStrategy {
    @Override
    public void buildRoute(String from, String to) {
        System.out.println("🚶 도보 경로 계산 중...");
        System.out.println("   출발: " + from);
        System.out.println("   도착: " + to);
        System.out.println("   경로: 인도 및 횡단보도 이용");
        System.out.println("   거리: 3 km");
    }
    
    @Override
    public int estimateTime(String from, String to) {
        return 40; // 40분
    }
}

/**
 * ConcreteStrategy: 대중교통 경로
 */
public class PublicTransportStrategy implements RouteStrategy {
    @Override
    public void buildRoute(String from, String to) {
        System.out.println("🚇 대중교통 경로 계산 중...");
        System.out.println("   출발: " + from);
        System.out.println("   도착: " + to);
        System.out.println("   경로: 지하철 2호선 → 버스 720번");
        System.out.println("   정류장: 5개");
    }
    
    @Override
    public int estimateTime(String from, String to) {
        return 50; // 50분
    }
}

/**
 * Context: 내비게이터
 */
public class Navigator {
    private RouteStrategy strategy;
    
    public void setStrategy(RouteStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void navigate(String from, String to) {
        if (strategy == null) {
            System.out.println("❌ 이동 수단을 선택해주세요!");
            return;
        }
        
        System.out.println("\n=== 경로 안내 ===");
        strategy.buildRoute(from, to);
        int time = strategy.estimateTime(from, to);
        System.out.println("   예상 시간: " + time + "분");
    }
}

/**
 * 사용 예제
 */
public class NavigationExample {
    public static void main(String[] args) {
        Navigator navigator = new Navigator();
        String from = "강남역";
        String to = "홍대입구역";
        
        // 자동차
        System.out.println("### 자동차 경로 ###");
        navigator.setStrategy(new CarRouteStrategy());
        navigator.navigate(from, to);
        
        // 도보
        System.out.println("\n### 도보 경로 ###");
        navigator.setStrategy(new WalkRouteStrategy());
        navigator.navigate(from, to);
        
        // 대중교통
        System.out.println("\n### 대중교통 경로 ###");
        navigator.setStrategy(new PublicTransportStrategy());
        navigator.navigate(from, to);
    }
}
```

---

### 예제 2: 데이터 압축 ⭐⭐⭐

```java
/**
 * Strategy: 압축 전략
 */
public interface CompressionStrategy {
    byte[] compress(byte[] data);
    byte[] decompress(byte[] data);
    String getAlgorithmName();
}

/**
 * ConcreteStrategy: ZIP 압축
 */
public class ZipCompressionStrategy implements CompressionStrategy {
    @Override
    public byte[] compress(byte[] data) {
        System.out.println("🗜️ ZIP 압축 중...");
        System.out.println("   원본 크기: " + data.length + " bytes");
        
        // 실제로는 ZIP 압축 수행
        byte[] compressed = new byte[data.length / 2];
        System.out.println("   압축 후: " + compressed.length + " bytes");
        System.out.println("   압축률: 50%");
        
        return compressed;
    }
    
    @Override
    public byte[] decompress(byte[] data) {
        System.out.println("📦 ZIP 압축 해제 중...");
        return new byte[data.length * 2];
    }
    
    @Override
    public String getAlgorithmName() {
        return "ZIP";
    }
}

/**
 * ConcreteStrategy: GZIP 압축
 */
public class GzipCompressionStrategy implements CompressionStrategy {
    @Override
    public byte[] compress(byte[] data) {
        System.out.println("🗜️ GZIP 압축 중...");
        System.out.println("   원본 크기: " + data.length + " bytes");
        
        // GZIP은 더 좋은 압축률
        byte[] compressed = new byte[data.length / 3];
        System.out.println("   압축 후: " + compressed.length + " bytes");
        System.out.println("   압축률: 67%");
        
        return compressed;
    }
    
    @Override
    public byte[] decompress(byte[] data) {
        System.out.println("📦 GZIP 압축 해제 중...");
        return new byte[data.length * 3];
    }
    
    @Override
    public String getAlgorithmName() {
        return "GZIP";
    }
}

/**
 * Context: 파일 압축기
 */
public class FileCompressor {
    private CompressionStrategy strategy;
    
    public FileCompressor(CompressionStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void setStrategy(CompressionStrategy strategy) {
        this.strategy = strategy;
        System.out.println("💡 압축 알고리즘 변경: " + strategy.getAlgorithmName());
    }
    
    public void compressFile(String fileName, byte[] data) {
        System.out.println("\n=== 파일 압축: " + fileName + " ===");
        byte[] compressed = strategy.compress(data);
        System.out.println("✅ 압축 완료!");
    }
    
    public void decompressFile(String fileName, byte[] data) {
        System.out.println("\n=== 파일 압축 해제: " + fileName + " ===");
        byte[] decompressed = strategy.decompress(data);
        System.out.println("✅ 압축 해제 완료!");
    }
}

/**
 * 사용 예제
 */
public class CompressionExample {
    public static void main(String[] args) {
        byte[] data = new byte[1000]; // 1000 bytes 데이터
        
        // ZIP 압축
        FileCompressor compressor = new FileCompressor(new ZipCompressionStrategy());
        compressor.compressFile("document.txt", data);
        
        // GZIP으로 변경
        compressor.setStrategy(new GzipCompressionStrategy());
        compressor.compressFile("document.txt", data);
    }
}
```

---

### 예제 3: 정렬 알고리즘 ⭐⭐

```java
/**
 * Strategy: 정렬 전략
 */
public interface SortStrategy {
    void sort(int[] array);
    String getAlgorithmName();
}

/**
 * ConcreteStrategy: 버블 정렬
 */
public class BubbleSortStrategy implements SortStrategy {
    @Override
    public void sort(int[] array) {
        System.out.println("🫧 버블 정렬 실행 중...");
        long start = System.nanoTime();
        
        for (int i = 0; i < array.length - 1; i++) {
            for (int j = 0; j < array.length - 1 - i; j++) {
                if (array[j] > array[j + 1]) {
                    int temp = array[j];
                    array[j] = array[j + 1];
                    array[j + 1] = temp;
                }
            }
        }
        
        long end = System.nanoTime();
        System.out.println("   실행 시간: " + (end - start) / 1000 + "μs");
    }
    
    @Override
    public String getAlgorithmName() {
        return "Bubble Sort";
    }
}

/**
 * ConcreteStrategy: 퀵 정렬
 */
public class QuickSortStrategy implements SortStrategy {
    @Override
    public void sort(int[] array) {
        System.out.println("⚡ 퀵 정렬 실행 중...");
        long start = System.nanoTime();
        
        quickSort(array, 0, array.length - 1);
        
        long end = System.nanoTime();
        System.out.println("   실행 시간: " + (end - start) / 1000 + "μs");
    }
    
    private void quickSort(int[] arr, int low, int high) {
        if (low < high) {
            int pi = partition(arr, low, high);
            quickSort(arr, low, pi - 1);
            quickSort(arr, pi + 1, high);
        }
    }
    
    private int partition(int[] arr, int low, int high) {
        int pivot = arr[high];
        int i = low - 1;
        
        for (int j = low; j < high; j++) {
            if (arr[j] < pivot) {
                i++;
                int temp = arr[i];
                arr[i] = arr[j];
                arr[j] = temp;
            }
        }
        
        int temp = arr[i + 1];
        arr[i + 1] = arr[high];
        arr[high] = temp;
        
        return i + 1;
    }
    
    @Override
    public String getAlgorithmName() {
        return "Quick Sort";
    }
}

/**
 * Context: 정렬기
 */
public class Sorter {
    private SortStrategy strategy;
    
    public void setStrategy(SortStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void sort(int[] array) {
        System.out.println("\n=== 정렬 실행: " + strategy.getAlgorithmName() + " ===");
        System.out.println("배열 크기: " + array.length);
        strategy.sort(array);
    }
}

/**
 * 사용 예제
 */
public class SortExample {
    public static void main(String[] args) {
        Sorter sorter = new Sorter();
        
        // 작은 배열: 버블 정렬
        int[] smallArray = {5, 2, 8, 1, 9};
        sorter.setStrategy(new BubbleSortStrategy());
        sorter.sort(smallArray);
        
        // 큰 배열: 퀵 정렬
        int[] largeArray = new int[1000];
        for (int i = 0; i < 1000; i++) {
            largeArray[i] = (int) (Math.random() * 1000);
        }
        
        sorter.setStrategy(new QuickSortStrategy());
        sorter.sort(largeArray);
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **OCP 준수** | 새 전략 추가 시 기존 코드 불변 | 새 결제 수단 |
| **알고리즘 교체** | 런타임에 동적 변경 | 압축 알고리즘 |
| **테스트 용이** | 각 전략 독립 테스트 | 단위 테스트 |
| **재사용성** | 전략을 여러 곳에서 사용 | 정렬 알고리즘 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **클래스 증가** | 전략마다 클래스 생성 | 익명 클래스/람다 |
| **전략 선택** | 클라이언트가 전략 알아야 함 | Factory 패턴 |

---

## 7. 안티패턴

### ❌ 안티패턴: 전략에 상태 저장

```java
// 잘못된 예: 전략이 상태를 가짐
public class BadStrategy implements Strategy {
    private int count = 0; // 상태!
    
    public void execute() {
        count++; // 위험!
    }
}
```

**해결:**
```java
// 전략은 무상태(Stateless)로
public class GoodStrategy implements Strategy {
    public void execute(Context context) {
        // context에서 필요한 데이터 받기
    }
}
```

---

## 8. 핵심 정리

### 📌 Strategy 패턴 체크리스트

```
✅ Strategy 인터페이스 정의
✅ ConcreteStrategy 구현
✅ Context 클래스 작성
✅ 전략 교체 메서드 제공
✅ 클라이언트는 전략 주입
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 알고리즘 교체 필요 | ⭐⭐⭐ | 런타임 변경 |
| if-else 많음 | ⭐⭐⭐ | 조건문 제거 |
| 알고리즘 독립 테스트 | ⭐⭐⭐ | 테스트 용이 |
| 여러 변형 알고리즘 | ⭐⭐⭐ | 캡슐화 |

### 💡 핵심 포인트

1. **알고리즘을 캡슐화**
2. **상속보다 조합**
3. **런타임에 교체 가능**
4. **if-else 제거**

### 🔥 실무 팁

```java
// Java 8+에서는 람다로 간단히
Context context = new Context();
context.setStrategy(data -> {
    // 전략 로직
});

// 또는 메서드 참조
context.setStrategy(this::processData);
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[다음: Observer →](02-Observer.md)**

</div>
