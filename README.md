<div align="center">

# 🎨 Java Design Patterns

**실전 예제로 배우는 디자인 패턴 완전 정복**

<br/>

> *"단순한 코드 작성을 넘어, 확장 가능하고 유지보수하기 쉬운 설계로"*

GoF 패턴부터 Modern Java 패턴까지,  
**왜 이렇게 설계하는지** 원리부터 파헤치는 디자인 패턴 심화 학습 자료

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-8%2B-orange?style=flat-square&logo=openjdk)](https://www.java.com)
[![Patterns](https://img.shields.io/badge/Patterns-47개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 프로젝트에 대하여

단순한 패턴 나열이 아닌, **실전에서 바로 적용할 수 있는** 디자인 패턴 가이드입니다.

### ✨ 특징

| 🎯 **문제 중심** | 💻 **실행 가능** | 🔥 **실전 사례** | 📊 **비교 분석** |
|:---:|:---:|:---:|:---:|
| 어떤 문제를<br/>해결하는지부터 | 모든 패턴은<br/>동작하는 코드로 | 실무에서<br/>자주 쓰는 사례 | 패턴 간<br/>장단점 비교 |

- ✅ **47가지 디자인 패턴** - GoF부터 아키텍처, 동시성, Modern Java 패턴까지
- ✅ **실행 가능한 예제 코드** - 이론만이 아닌 실습 중심
- ✅ **문제-해결 구조** - 패턴이 필요한 이유부터 명확히
- ✅ **Before/After 비교** - 패턴 적용 전후 비교
- ✅ **안티패턴 가이드** - 잘못된 사용 사례와 해결책

---

## 📚 목차

> 💡 **각 챕터를 클릭하면 상세한 학습 문서로 이동합니다**

### 🔹 1. 생성 패턴 (Creational Patterns)
객체 생성 메커니즘을 다루는 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:-------:|----------|-----------|
| **[01. Singleton](./creational/Singleton.md)** | 전역 인스턴스가 필요할 때 | Thread-safe, Enum, Lazy Initialization |
| **[02. Factory Method](./creational/FactoryMethod.md)** | 객체 생성 로직을 분리하고 싶을 때 | 인터페이스 기반, 확장성 |
| **[03. Abstract Factory](./creational/AbstractFactory.md)** | 관련 객체군을 생성할 때 | 제품군, 플랫폼 독립 |
| **[04. Builder](./creational/Builder.md)** | 복잡한 객체를 단계별로 생성할 때 | Fluent API, Immutable |
| **[05. Prototype](./creational/Prototype.md)** | 객체를 복제해서 생성할 때 | Clone, Deep Copy |
| **[06. Object Pool](./creational/ObjectPool.md)** | 비용이 큰 객체를 재사용할 때 | Connection Pool, Thread Pool |

<br/>

### 🔹 2. 구조 패턴 (Structural Patterns)
클래스와 객체를 조합하는 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:-------:|----------|-----------|
| **[07. Adapter](./structural/Adapter.md)** | 호환되지 않는 인터페이스를 연결할 때 | Wrapper, 레거시 통합 |
| **[08. Bridge](./structural/Bridge.md)** | 추상화와 구현을 분리할 때 | 다차원 확장, 독립적 변경 |
| **[09. Composite](./structural/Composite.md)** | 트리 구조로 객체를 다룰 때 | 부분-전체, 재귀적 구조 |
| **[10. Decorator](./structural/Decorator.md)** | 동적으로 기능을 추가할 때 | 상속 대신 조합, Wrapping |
| **[11. Facade](./structural/Facade.md)** | 복잡한 서브시스템을 단순화할 때 | 통합 인터페이스, 결합도 감소 |
| **[12. Flyweight](./structural/Flyweight.md)** | 많은 객체를 효율적으로 관리할 때 | 공유, 메모리 절약 |
| **[13. Proxy](./structural/Proxy.md)** | 객체 접근을 제어할 때 | Lazy Loading, 권한 체크 |

<br/>

### 🔹 3. 행위 패턴 (Behavioral Patterns)
객체 간 책임과 알고리즘 분배 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:-------:|----------|-----------|
| **[14. Chain of Responsibility](./behavioral/ChainOfResponsibility.md)** | 요청을 순차적으로 처리할 때 | 책임 연쇄, 파이프라인 |
| **[15. Command](./behavioral/Command.md)** | 요청을 객체로 캡슐화할 때 | Undo/Redo, 트랜잭션 |
| **[16. Interpreter](./behavioral/Interpreter.md)** | 언어 문법을 해석할 때 | AST, 파서 |
| **[17. Iterator](./behavioral/Iterator.md)** | 컬렉션을 순회할 때 | 내부 구조 감춤, 표준화 |
| **[18. Mediator](./behavioral/Mediator.md)** | 객체 간 상호작용을 중재할 때 | 결합도 감소, 중앙 제어 |
| **[19. Memento](./behavioral/Memento.md)** | 객체 상태를 저장/복원할 때 | 스냅샷, 이력 관리 |
| **[20. Observer](./behavioral/Observer.md)** | 이벤트를 구독/발행할 때 | 일대다 의존, Pub/Sub |
| **[21. State](./behavioral/State.md)** | 상태에 따라 행위가 변할 때 | 상태 기계, 조건문 제거 |
| **[22. Strategy](./behavioral/Strategy.md)** | 알고리즘을 교체 가능하게 할 때 | 정책 분리, 런타임 교체 |
| **[23. Template Method](./behavioral/TemplateMethod.md)** | 알고리즘 골격을 정의할 때 | Hook 메서드, 제어 역전 |
| **[24. Visitor](./behavioral/Visitor.md)** | 구조와 연산을 분리할 때 | Double Dispatch, 확장성 |

<br/>

### 🔹 4. 아키텍처 패턴 (Architectural Patterns)
시스템 레벨 구조 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:-------:|----------|-----------|
| **[25. MVC](./architectural/MVC.md)** | UI 로직을 분리할 때 | Model-View-Controller |
| **[26. MVP](./architectural/MVP.md)** | View를 수동적으로 만들 때 | Presenter, Testability |
| **[27. MVVM](./architectural/MVVM.md)** | 데이터 바인딩이 필요할 때 | ViewModel, Reactive |
| **[28. Layered Architecture](./architectural/Layered.md)** | 계층을 분리할 때 | Presentation-Business-Data |
| **[29. Hexagonal](./architectural/Hexagonal.md)** | 도메인을 격리할 때 | Ports & Adapters, DDD |
| **[30. Event-Driven](./architectural/EventDriven.md)** | 비동기 통신이 필요할 때 | Message Queue, 느슨한 결합 |
| **[31. Microservices](./architectural/Microservices.md)** | 독립 배포가 필요할 때 | Service Mesh, API Gateway |
| **[32. Repository](./architectural/Repository.md)** | 데이터 접근을 추상화할 때 | Domain Model 보호 |

<br/>

### 🔹 5. 동시성 패턴 (Concurrency Patterns)
멀티스레딩 환경의 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:-------:|----------|-----------|
| **[33. Thread Pool](./concurrency/ThreadPool.md)** | 스레드를 효율적으로 관리할 때 | ExecutorService, 재사용 |
| **[34. Producer-Consumer](./concurrency/ProducerConsumer.md)** | 생산/소비를 분리할 때 | BlockingQueue, 버퍼 |
| **[35. Reader-Writer Lock](./concurrency/ReaderWriterLock.md)** | 읽기/쓰기를 분리할 때 | 동시 읽기, 배타적 쓰기 |
| **[36. Double-Checked Locking](./concurrency/DoubleCheckedLocking.md)** | Singleton을 최적화할 때 | volatile, 성능 |
| **[37. Active Object](./concurrency/ActiveObject.md)** | 비동기 메서드를 호출할 때 | Actor Model, 메시지 |
| **[38. Future/Promise](./concurrency/FuturePromise.md)** | 비동기 결과를 처리할 때 | CompletableFuture, 콜백 |

<br/>

### 🔹 6. 엔터프라이즈 패턴 (Enterprise Patterns)
비즈니스 로직 구현 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:-------:|----------|-----------|
| **[39. DTO](./enterprise/DTO.md)** | 계층 간 데이터를 전송할 때 | 직렬화, API 응답 |
| **[40. DAO](./enterprise/DAO.md)** | DB 접근을 캡슐화할 때 | CRUD, SQL 격리 |
| **[41. Service Layer](./enterprise/ServiceLayer.md)** | 비즈니스 로직을 분리할 때 | Transaction, Orchestration |
| **[42. Unit of Work](./enterprise/UnitOfWork.md)** | 트랜잭션을 관리할 때 | 변경 추적, 일괄 처리 |
| **[43. Specification](./enterprise/Specification.md)** | 비즈니스 규칙을 캡슐화할 때 | 조합 가능, 재사용 |

<br/>

### 🔹 7. Modern Java 패턴 (Modern Java Patterns)
Java 8+ 기능 활용 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:-------:|----------|-----------|
| **[44. Functional Interface](./modern/FunctionalInterface.md)** | 함수를 일급 객체로 다룰 때 | Lambda, Method Reference |
| **[45. Stream Pipeline](./modern/StreamPipeline.md)** | 데이터를 선언적으로 처리할 때 | filter-map-collect, 지연 평가 |
| **[46. Optional Chaining](./modern/OptionalChaining.md)** | Null을 안전하게 처리할 때 | ofNullable, orElse |
| **[47. Sealed Classes](./modern/SealedClasses.md)** | 타입을 제한할 때 | Pattern Matching, 타입 안전 |

<br/>

---

## 🗺️ 학습 로드맵

### 🎯 목적별 학습 경로

<details>
<summary><b>📘 입문자 (디자인 패턴 처음 배우는 분)</b></summary>

<br/>

**1주차: 생성 패턴 기초**
```
✅ Singleton - 가장 기본적인 패턴
✅ Factory Method - 객체 생성 분리
✅ Builder - 복잡한 객체 생성
```

**2주차: 구조 패턴 핵심**
```
✅ Adapter - 인터페이스 변환
✅ Decorator - 기능 추가
✅ Facade - 서브시스템 단순화
```

**3주차: 행위 패턴 기초**
```
✅ Strategy - 알고리즘 교체
✅ Observer - 이벤트 처리
✅ Template Method - 알고리즘 골격
```

**4주차: 실전 적용**
```
✅ Command - 요청 캡슐화
✅ State - 상태 기반 행위
✅ 실전 프로젝트 적용
```

</details>

<details>
<summary><b>💼 실무자 (프로젝트에 바로 적용)</b></summary>

<br/>

**Week 1: 핵심 GoF 패턴**
```
✅ Singleton, Factory, Builder
✅ Strategy, Observer, Template Method
✅ Adapter, Decorator, Proxy
```

**Week 2: 아키텍처 패턴**
```
✅ Layered Architecture
✅ Hexagonal (Ports & Adapters)
✅ Repository, Service Layer
```

**Week 3: 엔터프라이즈 패턴**
```
✅ DTO, DAO, Service Layer
✅ Unit of Work, Specification
✅ 트랜잭션 관리
```

**Week 4: 동시성 & Modern Java**
```
✅ Thread Pool, Producer-Consumer
✅ CompletableFuture 패턴
✅ Functional Interface, Stream Pipeline
```

</details>

<details>
<summary><b>🏆 면접 준비</b></summary>

<br/>

**우선순위 1 (필수):**
```
✅ Singleton - 구현 방법 3가지
✅ Factory vs Abstract Factory 차이
✅ Strategy vs Template Method 차이
✅ Decorator vs Proxy 차이
```

**우선순위 2 (중요):**
```
✅ Builder 패턴 장점
✅ Observer 패턴 구현
✅ MVC vs MVP vs MVVM
✅ Repository 패턴 필요성
```

**우선순위 3 (심화):**
```
✅ Visitor 패턴 Double Dispatch
✅ Flyweight 메모리 최적화
✅ Hexagonal Architecture
✅ SOLID 원칙과 패턴의 관계
```

</details>

<details>
<summary><b>🚀 아키텍트 지향</b></summary>

<br/>

**Phase 1: GoF 패턴 마스터**
```
✅ 23가지 GoF 패턴 전체
✅ 패턴 간 관계와 조합
✅ 적용 시기와 트레이드오프
```

**Phase 2: 아키텍처 패턴**
```
✅ MVC, MVP, MVVM
✅ Layered, Hexagonal
✅ Event-Driven, Microservices
```

**Phase 3: 엔터프라이즈 패턴**
```
✅ DDD와 패턴
✅ CQRS, Event Sourcing
✅ 분산 시스템 패턴
```

**Phase 4: 실전 설계**
```
✅ 패턴 조합 전략
✅ 리팩토링과 패턴
✅ 레거시 코드 개선
```

</details>

<details>
<summary><b>⚡ 빠른 복습 (경력 개발자)</b></summary>

<br/>

**Day 1: 생성 & 구조 패턴**
```
✅ Singleton, Factory, Builder
✅ Adapter, Decorator, Proxy, Facade
```

**Day 2: 행위 & 아키텍처 패턴**
```
✅ Strategy, Observer, Template Method, Command
✅ MVC, Layered, Repository
```

**Day 3: 동시성 & Modern Java**
```
✅ Thread Pool, CompletableFuture
✅ Functional Interface, Stream Pipeline
```

</details>

---

## 🎓 학습 방법

```
📖 Read → 💻 Practice → 🤔 Think → 📝 Review → 🔁 Repeat
```

### 1️⃣ 기초부터 차근차근
```
생성 패턴 (Singleton → Factory → Builder) 
→ 구조 패턴 (Adapter → Decorator) 
→ 행위 패턴 (Strategy → Observer)
```

### 2️⃣ 실무 중심 학습
```
실제 프로젝트 문제 파악 
→ 적합한 패턴 선택 
→ Before/After 비교 
→ 리팩토링 적용
```

### 3️⃣ 패턴 조합 연습
```
단일 패턴 익히기 
→ 패턴 간 관계 이해 
→ 패턴 조합 설계 
→ 복잡한 시스템 설계
```

---

## 💻 시작하기

### 📋 필요 사항
- **Java 8** 이상 (일부 Java 11+ 기능 포함)
- **IntelliJ IDEA** 또는 Eclipse, VS Code

### 1️⃣ Repository 클론
```bash
git clone https://github.com/dev-book-lab/java-design-patterns.git
cd java-design-patterns
```

### 2️⃣ 학습 방법
1. 문제 상황을 먼저 이해하기
2. 패턴 없는 코드(Before) 확인
3. 패턴 적용 코드(After) 비교
4. 예제 코드를 직접 실행하며 실습
5. 자신의 프로젝트에 적용해보기

---

## 📖 문서 구성

각 문서는 다음과 같은 구조로 구성됩니다:

| 섹션 | 설명 |
|------|------|
| 🎯 **문제 상황** | 패턴이 필요한 이유 |
| 📌 **핵심 개념** | 패턴의 구조와 원리 |
| 💻 **구현 예제** | Before/After 비교 코드 |
| 🔥 **실전 사례** | 실무 적용 예제 |
| ⚡ **장단점** | 패턴의 트레이드오프 |
| 🚫 **안티패턴** | 잘못된 사용 사례 |
| 📌 **핵심 정리** | 빠른 복습용 요약 |

---

## 🤝 기여하기

더 좋은 예제나 설명이 있다면 언제든 환영합니다!

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingPattern`)
3. Commit your changes (`git commit -m 'Add amazing pattern example'`)
4. Push to the branch (`git push origin feature/AmazingPattern`)
5. Open a Pull Request

---

## 🙏 Reference

- [Design Patterns: Elements of Reusable Object-Oriented Software (GoF)](https://en.wikipedia.org/wiki/Design_Patterns)
- [Head First Design Patterns](https://www.oreilly.com/library/view/head-first-design/0596007124/)
- [Refactoring Guru - Design Patterns](https://refactoring.guru/design-patterns)

---

## ✨ Dev Book Lab

<div align="center">

**AI와 함께 개발 서적을 분석하고 정리하는 연구소**

[📂 More Projects](https://github.com/dev-book-lab)

</div>

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by Dev Book Lab

<br/>

*"단순한 코드 작성을 넘어, 확장 가능하고 유지보수하기 쉬운 설계로"*

</div>
