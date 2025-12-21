# Template Method Pattern (템플릿 메서드 패턴)

> **"알고리즘의 골격을 정의하고 세부 구현은 서브클래스에 맡기자"**

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
// 문제 1: 중복된 알고리즘 골격
public class TeaMaker {
    public void makeTea() {
        boilWater();         // 1. 물 끓이기
        steepTeaBag();       // 2. 차 우리기 (다름!)
        pourInCup();         // 3. 컵에 따르기
        addLemon();          // 4. 레몬 추가 (다름!)
    }
}

public class CoffeeMaker {
    public void makeCoffee() {
        boilWater();         // 1. 물 끓이기 (중복!)
        brewCoffeeGrinds();  // 2. 커피 내리기 (다름!)
        pourInCup();         // 3. 컵에 따르기 (중복!)
        addSugarAndMilk();   // 4. 설탕과 우유 (다름!)
    }
}
// 공통 단계는 중복, 다른 단계만 재정의하고 싶음!

// 문제 2: 알고리즘 순서 강제 불가
public class DataProcessor {
    public void process() {
        // 개발자가 순서를 실수할 수 있음
        readData();
        processData();
        saveData();
        // 또는
        readData();
        saveData();    // 처리 안 하고 저장? 버그!
        processData();
    }
}

// 문제 3: 공통 로직 변경이 어려움
public class FileReader1 {
    public void readFile() {
        openFile();
        readData();
        closeFile();
    }
}

public class FileReader2 {
    public void readFile() {
        openFile();
        readData();
        closeFile();
    }
}
// openFile(), closeFile() 로직 변경 시 모든 클래스 수정!

// 문제 4: 선택적 단계 구현
public class GameAI {
    public void turn() {
        collectResources();
        buildStructures();
        buildUnits();
        attack();
        
        // 어떤 AI는 공격 안 하고 싶은데...
        // 빈 메서드로 만들어야 함?
    }
}
```

### ⚡ 핵심 문제

1. **코드 중복**: 알고리즘 골격이 여러 곳에 중복
2. **순서 강제 불가**: 단계 순서를 보장할 수 없음
3. **변경 어려움**: 공통 로직 변경 시 여러 곳 수정
4. **선택적 구현**: 일부 단계만 재정의하기 어려움

---

## 2. 패턴 정의

### 📖 정의

> **알고리즘의 구조를 메서드에 정의하고, 하위 클래스에서 알고리즘 구조의 변경 없이 알고리즘의 특정 단계를 재정의하는 패턴**

### 🎯 목적

- **알고리즘 골격**: 상위 클래스에서 정의
- **단계 재정의**: 하위 클래스에서 세부 구현
- **코드 재사용**: 공통 로직 한 곳에
- **할리우드 원칙**: "우리가 호출할게, 당신이 호출하지 마!"

### 💡 핵심 아이디어

```java
// Before: 중복된 골격
class Tea {
    void make() {
        boilWater();
        steepTea();    // 다름
        pourInCup();
        addLemon();    // 다름
    }
}

class Coffee {
    void make() {
        boilWater();   // 중복
        brewCoffee();  // 다름
        pourInCup();   // 중복
        addMilk();     // 다름
    }
}

// After: 템플릿 메서드
abstract class Beverage {
    // 템플릿 메서드 (final로 변경 불가)
    final void make() {
        boilWater();     // 공통
        brew();          // 하위 클래스 구현
        pourInCup();     // 공통
        addCondiments(); // 하위 클래스 구현
    }
    
    abstract void brew();
    abstract void addCondiments();
}
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌──────────────────────┐
│  AbstractClass       │  ← 추상 클래스
├──────────────────────┤
│ + templateMethod()   │  ← 템플릿 메서드 (final)
│ # primitiveOp1()     │  ← 추상 메서드
│ # primitiveOp2()     │  ← 추상 메서드
│ # hook()             │  ← 훅 메서드 (선택)
└──────────────────────┘
           △
           │ extends
┌──────────────────────┐
│  ConcreteClass       │
├──────────────────────┤
│ # primitiveOp1()     │  ← 구체적 구현
│ # primitiveOp2()     │  ← 구체적 구현
└──────────────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **AbstractClass** | 템플릿 메서드 정의 | `Beverage` |
| **templateMethod()** | 알고리즘 골격 (final) | `make()` |
| **primitiveOp()** | 하위 클래스 구현 | `brew()` |
| **hook()** | 선택적 재정의 | `customerWantsCondiments()` |
| **ConcreteClass** | 세부 구현 | `Tea`, `Coffee` |

---

## 4. 구현 방법

### 기본 구현: 음료 제조 시스템 ⭐⭐⭐

```java
/**
 * AbstractClass: 음료 추상 클래스
 */
public abstract class Beverage {
    
    /**
     * 템플릿 메서드 (final - 변경 불가)
     * 알고리즘의 골격을 정의
     */
    public final void prepareRecipe() {
        System.out.println("\n=== " + getName() + " 제조 시작 ===");
        
        boilWater();
        brew();
        pourInCup();
        
        if (customerWantsCondiments()) { // 훅 메서드
            addCondiments();
        }
        
        System.out.println("=== 완성! ===\n");
    }
    
    /**
     * 공통 메서드
     */
    private void boilWater() {
        System.out.println("1️⃣ 물 끓이는 중...");
    }
    
    private void pourInCup() {
        System.out.println("3️⃣ 컵에 따르는 중...");
    }
    
    /**
     * 추상 메서드 - 하위 클래스에서 반드시 구현
     */
    protected abstract void brew();
    protected abstract void addCondiments();
    protected abstract String getName();
    
    /**
     * 훅 메서드 - 하위 클래스에서 선택적으로 오버라이드
     */
    protected boolean customerWantsCondiments() {
        return true; // 기본값
    }
}

/**
 * ConcreteClass 1: 차
 */
public class Tea extends Beverage {
    
    @Override
    protected void brew() {
        System.out.println("2️⃣ 차를 우려내는 중...");
    }
    
    @Override
    protected void addCondiments() {
        System.out.println("4️⃣ 레몬을 추가하는 중...");
    }
    
    @Override
    protected String getName() {
        return "홍차";
    }
}

/**
 * ConcreteClass 2: 커피
 */
public class Coffee extends Beverage {
    
    @Override
    protected void brew() {
        System.out.println("2️⃣ 커피를 내리는 중...");
    }
    
    @Override
    protected void addCondiments() {
        System.out.println("4️⃣ 설탕과 우유를 추가하는 중...");
    }
    
    @Override
    protected String getName() {
        return "커피";
    }
}

/**
 * ConcreteClass 3: 블랙커피 (훅 메서드 활용)
 */
public class BlackCoffee extends Beverage {
    
    @Override
    protected void brew() {
        System.out.println("2️⃣ 에스프레소를 추출하는 중...");
    }
    
    @Override
    protected void addCondiments() {
        System.out.println("4️⃣ 추가 없음 (블랙)");
    }
    
    @Override
    protected String getName() {
        return "블랙커피";
    }
    
    /**
     * 훅 메서드 오버라이드 - 첨가물 안 넣음
     */
    @Override
    protected boolean customerWantsCondiments() {
        return false; // 첨가물 안 넣음
    }
}

/**
 * 사용 예제
 */
public class TemplateMethodExample {
    public static void main(String[] args) {
        // 차 만들기
        Beverage tea = new Tea();
        tea.prepareRecipe();
        
        // 커피 만들기
        Beverage coffee = new Coffee();
        coffee.prepareRecipe();
        
        // 블랙커피 만들기 (첨가물 없음)
        Beverage blackCoffee = new BlackCoffee();
        blackCoffee.prepareRecipe();
    }
}
```

**실행 결과:**
```
=== 홍차 제조 시작 ===
1️⃣ 물 끓이는 중...
2️⃣ 차를 우려내는 중...
3️⃣ 컵에 따르는 중...
4️⃣ 레몬을 추가하는 중...
=== 완성! ===


=== 커피 제조 시작 ===
1️⃣ 물 끓이는 중...
2️⃣ 커피를 내리는 중...
3️⃣ 컵에 따르는 중...
4️⃣ 설탕과 우유를 추가하는 중...
=== 완성! ===


=== 블랙커피 제조 시작 ===
1️⃣ 물 끓이는 중...
2️⃣ 에스프레소를 추출하는 중...
3️⃣ 컵에 따르는 중...
=== 완성! ===
```

---

## 5. 실전 예제

### 예제 1: 데이터 마이닝 ⭐⭐⭐

```java
/**
 * AbstractClass: 데이터 마이너
 */
public abstract class DataMiner {
    
    /**
     * 템플릿 메서드
     */
    public final void mine(String path) {
        System.out.println("\n=== 데이터 마이닝 시작 ===");
        System.out.println("파일: " + path);
        
        Object file = openFile(path);
        Object rawData = extractData(file);
        Object data = parseData(rawData);
        Object analysis = analyzeData(data);
        sendReport(analysis);
        closeFile(file);
        
        System.out.println("=== 완료 ===\n");
    }
    
    // 추상 메서드 - 하위 클래스에서 구현
    protected abstract Object openFile(String path);
    protected abstract Object extractData(Object file);
    protected abstract Object parseData(Object rawData);
    
    // 공통 메서드
    protected Object analyzeData(Object data) {
        System.out.println("4️⃣ 데이터 분석 중...");
        return "분석 결과";
    }
    
    protected void sendReport(Object analysis) {
        System.out.println("5️⃣ 리포트 전송 중...");
    }
    
    protected void closeFile(Object file) {
        System.out.println("6️⃣ 파일 닫는 중...");
    }
}

/**
 * ConcreteClass: PDF 데이터 마이너
 */
public class PDFDataMiner extends DataMiner {
    
    @Override
    protected Object openFile(String path) {
        System.out.println("1️⃣ PDF 파일 열기: " + path);
        return "PDF File Object";
    }
    
    @Override
    protected Object extractData(Object file) {
        System.out.println("2️⃣ PDF에서 텍스트 추출 중...");
        return "Raw PDF Text";
    }
    
    @Override
    protected Object parseData(Object rawData) {
        System.out.println("3️⃣ PDF 텍스트 파싱 중...");
        return "Parsed PDF Data";
    }
}

/**
 * ConcreteClass: CSV 데이터 마이너
 */
public class CSVDataMiner extends DataMiner {
    
    @Override
    protected Object openFile(String path) {
        System.out.println("1️⃣ CSV 파일 열기: " + path);
        return "CSV File Object";
    }
    
    @Override
    protected Object extractData(Object file) {
        System.out.println("2️⃣ CSV 행 읽기 중...");
        return "Raw CSV Rows";
    }
    
    @Override
    protected Object parseData(Object rawData) {
        System.out.println("3️⃣ CSV 데이터 파싱 중...");
        return "Parsed CSV Data";
    }
}

/**
 * ConcreteClass: Excel 데이터 마이너
 */
public class ExcelDataMiner extends DataMiner {
    
    @Override
    protected Object openFile(String path) {
        System.out.println("1️⃣ Excel 파일 열기: " + path);
        return "Excel File Object";
    }
    
    @Override
    protected Object extractData(Object file) {
        System.out.println("2️⃣ Excel 시트 읽기 중...");
        return "Raw Excel Data";
    }
    
    @Override
    protected Object parseData(Object rawData) {
        System.out.println("3️⃣ Excel 데이터 파싱 중...");
        return "Parsed Excel Data";
    }
}

/**
 * 사용 예제
 */
public class DataMiningExample {
    public static void main(String[] args) {
        DataMiner pdfMiner = new PDFDataMiner();
        pdfMiner.mine("/data/report.pdf");
        
        DataMiner csvMiner = new CSVDataMiner();
        csvMiner.mine("/data/sales.csv");
        
        DataMiner excelMiner = new ExcelDataMiner();
        excelMiner.mine("/data/budget.xlsx");
    }
}
```

---

### 예제 2: 게임 AI ⭐⭐⭐

```java
/**
 * AbstractClass: 게임 AI
 */
public abstract class GameAI {
    
    /**
     * 템플릿 메서드
     */
    public final void turn() {
        System.out.println("\n=== " + getAIName() + " 턴 시작 ===");
        
        collectResources();
        buildStructures();
        buildUnits();
        
        if (shouldAttack()) { // 훅 메서드
            attack();
        } else {
            System.out.println("5️⃣ 방어 태세 유지");
        }
        
        System.out.println("=== 턴 종료 ===\n");
    }
    
    // 공통 메서드
    protected void collectResources() {
        System.out.println("1️⃣ 자원 수집");
    }
    
    // 추상 메서드
    protected abstract void buildStructures();
    protected abstract void buildUnits();
    protected abstract void attack();
    protected abstract String getAIName();
    
    // 훅 메서드
    protected boolean shouldAttack() {
        return true; // 기본: 공격
    }
}

/**
 * ConcreteClass: 공격형 AI
 */
public class AggressiveAI extends GameAI {
    
    @Override
    protected void buildStructures() {
        System.out.println("2️⃣ 병영 건설");
    }
    
    @Override
    protected void buildUnits() {
        System.out.println("3️⃣ 전투 유닛 대량 생산");
    }
    
    @Override
    protected void attack() {
        System.out.println("4️⃣ 전면 공격!");
    }
    
    @Override
    protected String getAIName() {
        return "공격형 AI";
    }
    
    @Override
    protected boolean shouldAttack() {
        return true; // 항상 공격
    }
}

/**
 * ConcreteClass: 방어형 AI
 */
public class DefensiveAI extends GameAI {
    
    @Override
    protected void buildStructures() {
        System.out.println("2️⃣ 방어 타워 건설");
    }
    
    @Override
    protected void buildUnits() {
        System.out.println("3️⃣ 방어 유닛 소량 생산");
    }
    
    @Override
    protected void attack() {
        System.out.println("4️⃣ 역습!");
    }
    
    @Override
    protected String getAIName() {
        return "방어형 AI";
    }
    
    @Override
    protected boolean shouldAttack() {
        // 랜덤으로 가끔만 공격
        return Math.random() > 0.7;
    }
}

/**
 * 사용 예제
 */
public class GameAIExample {
    public static void main(String[] args) {
        GameAI aggressive = new AggressiveAI();
        aggressive.turn();
        
        GameAI defensive = new DefensiveAI();
        defensive.turn();
        defensive.turn();
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **코드 재사용** | 공통 로직 한 곳에 | boilWater() |
| **알고리즘 제어** | 순서 강제 가능 | final 메서드 |
| **유연성** | 특정 단계만 재정의 | brew() |
| **할리우드 원칙** | 제어 역전 | 상위가 호출 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **상속 강제** | 상속 구조 필요 | 조합 고려 |
| **LSP 위반 가능** | 잘못된 재정의 시 | 명확한 계약 |
| **클래스 증가** | 변형마다 클래스 | 필요시에만 |

---

## 7. 안티패턴

### ❌ 안티패턴: 템플릿 메서드를 final로 안 함

```java
// 잘못된 예
public class Bad {
    public void templateMethod() { // final 없음!
        step1();
        step2();
    }
}

// 하위 클래스가 순서 바꿀 수 있음
class BadSub extends Bad {
    @Override
    public void templateMethod() {
        step2(); // 순서 변경!
        step1();
    }
}
```

**해결:**
```java
public final void templateMethod() { // final!
    step1();
    step2();
}
```

---

## 8. 핵심 정리

### 📌 Template Method 패턴 체크리스트

```
✅ AbstractClass 정의
✅ templateMethod() 작성 (final)
✅ 추상 메서드 선언
✅ 공통 메서드 구현
✅ 훅 메서드 제공 (선택)
✅ ConcreteClass 구현
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 알고리즘 골격 공통 | ⭐⭐⭐ | 코드 재사용 |
| 순서 강제 필요 | ⭐⭐⭐ | final 메서드 |
| 부분만 재정의 | ⭐⭐⭐ | 유연성 |
| 할리우드 원칙 | ⭐⭐⭐ | 제어 역전 |

### 💡 핵심 포인트

1. **알고리즘 골격 정의**
2. **final로 변경 방지**
3. **추상 메서드로 위임**
4. **훅으로 확장 포인트**

### 🔥 실무 활용

```java
// Java에서 흔히 볼 수 있음
// - AbstractList
// - HttpServlet (doGet, doPost)
// - JUnit (setUp, tearDown)
// - Spring Template (JdbcTemplate)
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Observer](02-Observer.md) | [다음: Command →](04-Command.md)**

</div>
