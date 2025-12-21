# Command Pattern (커맨드 패턴)

> **"요청을 객체로 캡슐화하여 실행을 지연하거나 취소하자"**

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
// 문제 1: 요청과 실행이 강하게 결합
public class TextEditor {
    public void saveButton() {
        save(); // 버튼이 직접 실행
    }
    
    public void copyButton() {
        copy(); // 버튼이 직접 실행
    }
    
    // 단축키도 추가하려면? 메뉴도?
    // 모든 곳에 save() 호출 반복!
}

// 문제 2: Undo/Redo 구현이 어려움
public class DrawingApp {
    public void drawLine() {
        // 선 그리기
        // 어떻게 취소할까?
        // 이전 상태를 어떻게 저장할까?
    }
    
    public void undo() {
        // ??? 무엇을 되돌릴까?
    }
}

// 문제 3: 작업 큐/로깅이 불가능
public class FileProcessor {
    public void processFile(String file) {
        // 파일 처리
        // 나중에 실행하려면?
        // 로그에 남기려면?
        // 재실행하려면?
    }
}

// 문제 4: 매크로/일괄 작업 어려움
public class ImageEditor {
    public void adjustBrightness() { }
    public void adjustContrast() { }
    public void applyFilter() { }
    
    // 이 작업들을 묶어서 "매크로"로 만들려면?
    // 한 번에 실행하려면?
}
```

### ⚡ 핵심 문제

1. **강한 결합**: 요청자와 수신자가 직접 연결
2. **Undo 불가**: 작업 취소 어려움
3. **작업 관리**: 큐잉, 로깅, 스케줄링 불가
4. **재사용 어려움**: 같은 작업을 다양한 곳에서 사용 어려움

---

## 2. 패턴 정의

### 📖 정의

> **요청을 객체로 캡슐화하여 서로 다른 요청으로 클라이언트를 매개변수화하고, 요청을 큐에 저장하거나 로그로 기록하며, 취소 가능한 연산을 지원하는 패턴**

### 🎯 목적

- **요청 객체화**: 요청을 객체로 만듦
- **디커플링**: 요청자와 수신자 분리
- **Undo/Redo**: 작업 취소/재실행
- **작업 관리**: 큐잉, 로깅, 스케줄링

### 💡 핵심 아이디어

```java
// Before: 직접 호출
button.onClick(() -> {
    editor.save(); // 강한 결합
});

// After: Command 객체
Command saveCommand = new SaveCommand(editor);
button.setCommand(saveCommand); // 약한 결합
button.click(); // command.execute()
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌─────────────┐
│   Client    │
└─────────────┘
       │ creates
       ▼
┌─────────────┐      ┌──────────────┐
│   Invoker   │      │   Command    │  ← 명령 인터페이스
├─────────────┤      ├──────────────┤
│ - command   │─────>│ + execute()  │
│ + execute() │      │ + undo()     │
└─────────────┘      └──────────────┘
                            △
                            │ implements
                   ┌────────┴────────┐
                   │                 │
          ┌────────────────┐ ┌───────────────┐
          │ConcreteCommand │ │ConcreteCommand│
          ├────────────────┤ ├───────────────┤
          │ - receiver     │ │ - receiver    │
          │ + execute()    │ │ + execute()   │
          │ + undo()       │ │ + undo()      │
          └────────────────┘ └───────────────┘
                   │                 │
                   │ calls           │
                   ▼                 ▼
          ┌────────────────┐
          │    Receiver    │  ← 실제 작업 수행
          ├────────────────┤
          │ + action()     │
          └────────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Command** | 명령 인터페이스 | `Command` |
| **ConcreteCommand** | 구체적 명령 | `SaveCommand` |
| **Receiver** | 실제 작업 수행 | `TextEditor` |
| **Invoker** | 명령 실행 | `Button` |
| **Client** | 명령 생성 | `Application` |

---

## 4. 구현 방법

### 기본 구현: 텍스트 에디터 ⭐⭐⭐

```java
/**
 * Command: 명령 인터페이스
 */
public interface Command {
    void execute();
    void undo();
}

/**
 * Receiver: 텍스트 에디터
 */
public class TextEditor {
    private StringBuilder text;
    
    public TextEditor() {
        this.text = new StringBuilder();
    }
    
    public void write(String words) {
        text.append(words);
        System.out.println("✍️ 입력: " + words);
    }
    
    public void deleteLast(int length) {
        if (text.length() >= length) {
            text.delete(text.length() - length, text.length());
            System.out.println("🗑️ 삭제: 마지막 " + length + "자");
        }
    }
    
    public void copy() {
        System.out.println("📋 복사: " + text.toString());
    }
    
    public void paste(String clipboard) {
        text.append(clipboard);
        System.out.println("📌 붙여넣기: " + clipboard);
    }
    
    public String getText() {
        return text.toString();
    }
    
    public void display() {
        System.out.println("📄 현재 텍스트: \"" + text.toString() + "\"");
    }
}

/**
 * ConcreteCommand 1: 쓰기 명령
 */
public class WriteCommand implements Command {
    private TextEditor editor;
    private String words;
    
    public WriteCommand(TextEditor editor, String words) {
        this.editor = editor;
        this.words = words;
    }
    
    @Override
    public void execute() {
        editor.write(words);
    }
    
    @Override
    public void undo() {
        editor.deleteLast(words.length());
    }
}

/**
 * ConcreteCommand 2: 복사 명령
 */
public class CopyCommand implements Command {
    private TextEditor editor;
    
    public CopyCommand(TextEditor editor) {
        this.editor = editor;
    }
    
    @Override
    public void execute() {
        editor.copy();
    }
    
    @Override
    public void undo() {
        System.out.println("↩️ 복사는 취소할 수 없습니다");
    }
}

/**
 * ConcreteCommand 3: 붙여넣기 명령
 */
public class PasteCommand implements Command {
    private TextEditor editor;
    private String clipboard;
    
    public PasteCommand(TextEditor editor, String clipboard) {
        this.editor = editor;
        this.clipboard = clipboard;
    }
    
    @Override
    public void execute() {
        editor.paste(clipboard);
    }
    
    @Override
    public void undo() {
        editor.deleteLast(clipboard.length());
    }
}

/**
 * Invoker: 명령 실행자 (히스토리 관리)
 */
public class CommandHistory {
    private Stack<Command> history;
    private Stack<Command> redoStack;
    
    public CommandHistory() {
        this.history = new Stack<>();
        this.redoStack = new Stack<>();
    }
    
    public void execute(Command command) {
        command.execute();
        history.push(command);
        redoStack.clear(); // 새 명령 실행 시 redo 스택 초기화
    }
    
    public void undo() {
        if (history.isEmpty()) {
            System.out.println("⚠️ 되돌릴 작업이 없습니다");
            return;
        }
        
        System.out.println("\n↩️ Undo 실행...");
        Command command = history.pop();
        command.undo();
        redoStack.push(command);
    }
    
    public void redo() {
        if (redoStack.isEmpty()) {
            System.out.println("⚠️ 다시 실행할 작업이 없습니다");
            return;
        }
        
        System.out.println("\n↪️ Redo 실행...");
        Command command = redoStack.pop();
        command.execute();
        history.push(command);
    }
    
    public void showHistory() {
        System.out.println("\n📚 명령 히스토리:");
        System.out.println("  History: " + history.size() + " 개");
        System.out.println("  Redo Stack: " + redoStack.size() + " 개");
    }
}

/**
 * 사용 예제
 */
public class CommandExample {
    public static void main(String[] args) {
        // Receiver 생성
        TextEditor editor = new TextEditor();
        
        // Invoker 생성
        CommandHistory history = new CommandHistory();
        
        // 작업 실행
        System.out.println("=== 텍스트 편집 시작 ===\n");
        
        history.execute(new WriteCommand(editor, "Hello"));
        editor.display();
        
        history.execute(new WriteCommand(editor, " World"));
        editor.display();
        
        history.execute(new CopyCommand(editor));
        
        history.execute(new PasteCommand(editor, "!!!"));
        editor.display();
        
        // Undo
        history.showHistory();
        
        history.undo();
        editor.display();
        
        history.undo();
        editor.display();
        
        // Redo
        history.redo();
        editor.display();
        
        history.showHistory();
    }
}
```

**실행 결과:**
```
=== 텍스트 편집 시작 ===

✍️ 입력: Hello
📄 현재 텍스트: "Hello"
✍️ 입력:  World
📄 현재 텍스트: "Hello World"
📋 복사: Hello World
📌 붙여넣기: !!!
📄 현재 텍스트: "Hello World!!!"

📚 명령 히스토리:
  History: 4 개
  Redo Stack: 0 개

↩️ Undo 실행...
🗑️ 삭제: 마지막 3자
📄 현재 텍스트: "Hello World"

↩️ Undo 실행...
↩️ 복사는 취소할 수 없습니다
📄 현재 텍스트: "Hello World"

↪️ Redo 실행...
📋 복사: Hello World
📄 현재 텍스트: "Hello World"

📚 명령 히스토리:
  History: 3 개
  Redo Stack: 1 개
```

---

## 5. 실전 예제

### 예제 1: 리모컨 시스템 ⭐⭐⭐

```java
/**
 * Receiver 1: TV
 */
public class TV {
    private boolean on = false;
    private int volume = 10;
    private int channel = 1;
    
    public void turnOn() {
        on = true;
        System.out.println("📺 TV 켜짐");
    }
    
    public void turnOff() {
        on = false;
        System.out.println("📺 TV 꺼짐");
    }
    
    public void volumeUp() {
        volume++;
        System.out.println("🔊 볼륨: " + volume);
    }
    
    public void volumeDown() {
        volume--;
        System.out.println("🔉 볼륨: " + volume);
    }
    
    public void setChannel(int channel) {
        this.channel = channel;
        System.out.println("📡 채널: " + channel);
    }
    
    public int getChannel() {
        return channel;
    }
    
    public int getVolume() {
        return volume;
    }
}

/**
 * Receiver 2: 조명
 */
public class Light {
    private boolean on = false;
    private int brightness = 50;
    
    public void turnOn() {
        on = true;
        System.out.println("💡 조명 켜짐");
    }
    
    public void turnOff() {
        on = false;
        System.out.println("💡 조명 꺼짐");
    }
    
    public void dim() {
        brightness = 30;
        System.out.println("🔅 조명 어둡게: " + brightness + "%");
    }
    
    public void brighten() {
        brightness = 100;
        System.out.println("🔆 조명 밝게: " + brightness + "%");
    }
    
    public int getBrightness() {
        return brightness;
    }
}

/**
 * ConcreteCommand: TV 켜기
 */
public class TVOnCommand implements Command {
    private TV tv;
    
    public TVOnCommand(TV tv) {
        this.tv = tv;
    }
    
    @Override
    public void execute() {
        tv.turnOn();
    }
    
    @Override
    public void undo() {
        tv.turnOff();
    }
}

/**
 * ConcreteCommand: 조명 켜기
 */
public class LightOnCommand implements Command {
    private Light light;
    
    public LightOnCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.turnOn();
    }
    
    @Override
    public void undo() {
        light.turnOff();
    }
}

/**
 * ConcreteCommand: 매크로 명령 (여러 명령 묶기)
 */
public class MacroCommand implements Command {
    private Command[] commands;
    
    public MacroCommand(Command[] commands) {
        this.commands = commands;
    }
    
    @Override
    public void execute() {
        System.out.println("🎬 매크로 실행...");
        for (Command command : commands) {
            command.execute();
        }
    }
    
    @Override
    public void undo() {
        System.out.println("↩️ 매크로 취소...");
        // 역순으로 취소
        for (int i = commands.length - 1; i >= 0; i--) {
            commands[i].undo();
        }
    }
}

/**
 * Invoker: 리모컨
 */
public class RemoteControl {
    private Command[] onCommands;
    private Command[] offCommands;
    private Command lastCommand;
    
    public RemoteControl() {
        onCommands = new Command[7];
        offCommands = new Command[7];
        
        Command noCommand = new Command() {
            public void execute() { }
            public void undo() { }
        };
        
        for (int i = 0; i < 7; i++) {
            onCommands[i] = noCommand;
            offCommands[i] = noCommand;
        }
        lastCommand = noCommand;
    }
    
    public void setCommand(int slot, Command onCommand, Command offCommand) {
        onCommands[slot] = onCommand;
        offCommands[slot] = offCommand;
    }
    
    public void onButtonPressed(int slot) {
        System.out.println("\n▶️ ON 버튼 " + slot + " 누름");
        onCommands[slot].execute();
        lastCommand = onCommands[slot];
    }
    
    public void offButtonPressed(int slot) {
        System.out.println("\n⏹️ OFF 버튼 " + slot + " 누름");
        offCommands[slot].execute();
        lastCommand = offCommands[slot];
    }
    
    public void undoButtonPressed() {
        System.out.println("\n↩️ UNDO 버튼 누름");
        lastCommand.undo();
    }
}

/**
 * 사용 예제
 */
public class RemoteControlExample {
    public static void main(String[] args) {
        // Receiver 생성
        TV tv = new TV();
        Light light = new Light();
        
        // Command 생성
        Command tvOn = new TVOnCommand(tv);
        Command tvOff = new Command() {
            public void execute() { tv.turnOff(); }
            public void undo() { tv.turnOn(); }
        };
        
        Command lightOn = new LightOnCommand(light);
        Command lightOff = new Command() {
            public void execute() { light.turnOff(); }
            public void undo() { light.turnOn(); }
        };
        
        // Invoker 생성 및 설정
        RemoteControl remote = new RemoteControl();
        remote.setCommand(0, tvOn, tvOff);
        remote.setCommand(1, lightOn, lightOff);
        
        // 사용
        System.out.println("=== 리모컨 사용 ===");
        remote.onButtonPressed(0);
        remote.onButtonPressed(1);
        
        remote.undoButtonPressed();
        
        // 매크로 (모든 기기 켜기)
        System.out.println("\n=== 매크로 사용 ===");
        Command[] partyOn = { tvOn, lightOn };
        Command partyMacro = new MacroCommand(partyOn);
        
        partyMacro.execute();
        partyMacro.undo();
    }
}
```

---

### 예제 2: 작업 큐 시스템 ⭐⭐⭐

```java
/**
 * Receiver: 파일 시스템
 */
public class FileSystem {
    public void createFile(String name) {
        System.out.println("📄 파일 생성: " + name);
    }
    
    public void deleteFile(String name) {
        System.out.println("🗑️ 파일 삭제: " + name);
    }
    
    public void copyFile(String source, String dest) {
        System.out.println("📋 파일 복사: " + source + " → " + dest);
    }
}

/**
 * ConcreteCommand: 파일 생성
 */
public class CreateFileCommand implements Command {
    private FileSystem fs;
    private String fileName;
    
    public CreateFileCommand(FileSystem fs, String fileName) {
        this.fs = fs;
        this.fileName = fileName;
    }
    
    @Override
    public void execute() {
        fs.createFile(fileName);
    }
    
    @Override
    public void undo() {
        fs.deleteFile(fileName);
    }
}

/**
 * ConcreteCommand: 파일 삭제
 */
public class DeleteFileCommand implements Command {
    private FileSystem fs;
    private String fileName;
    
    public DeleteFileCommand(FileSystem fs, String fileName) {
        this.fs = fs;
        this.fileName = fileName;
    }
    
    @Override
    public void execute() {
        fs.deleteFile(fileName);
    }
    
    @Override
    public void undo() {
        fs.createFile(fileName);
    }
}

/**
 * Invoker: 작업 큐
 */
public class TaskQueue {
    private Queue<Command> queue;
    
    public TaskQueue() {
        this.queue = new LinkedList<>();
    }
    
    public void addTask(Command command) {
        queue.offer(command);
        System.out.println("➕ 작업 추가 (큐 크기: " + queue.size() + ")");
    }
    
    public void processAll() {
        System.out.println("\n⚙️ 모든 작업 처리 시작...");
        
        while (!queue.isEmpty()) {
            Command command = queue.poll();
            command.execute();
        }
        
        System.out.println("✅ 모든 작업 완료!");
    }
    
    public int size() {
        return queue.size();
    }
}

/**
 * 사용 예제
 */
public class TaskQueueExample {
    public static void main(String[] args) {
        FileSystem fs = new FileSystem();
        TaskQueue queue = new TaskQueue();
        
        // 작업 추가
        System.out.println("=== 작업 큐에 추가 ===");
        queue.addTask(new CreateFileCommand(fs, "file1.txt"));
        queue.addTask(new CreateFileCommand(fs, "file2.txt"));
        queue.addTask(new DeleteFileCommand(fs, "old.txt"));
        queue.addTask(new CreateFileCommand(fs, "file3.txt"));
        
        // 일괄 처리
        queue.processAll();
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **디커플링** | 요청자와 수신자 분리 | 리모컨과 TV |
| **Undo/Redo** | 작업 취소/재실행 | 텍스트 에디터 |
| **큐잉** | 작업 대기열 관리 | 작업 큐 |
| **로깅** | 명령 기록 가능 | 감사 로그 |
| **매크로** | 여러 명령 조합 | 파티 모드 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **클래스 증가** | 명령마다 클래스 | 익명 클래스/람다 |
| **복잡도** | 단순 작업에 과함 | 필요시에만 |

---

## 7. 안티패턴

### ❌ 안티패턴: Command에 비즈니스 로직

```java
// 잘못된 예
public class BadCommand implements Command {
    public void execute() {
        // 복잡한 비즈니스 로직
        // Command는 단순히 위임만!
    }
}
```

**해결:**
```java
public class GoodCommand implements Command {
    private Receiver receiver;
    
    public void execute() {
        receiver.action(); // 위임!
    }
}
```

---

## 8. 핵심 정리

### 📌 Command 패턴 체크리스트

```
✅ Command 인터페이스 정의
✅ ConcreteCommand 구현
✅ Receiver 정의
✅ Invoker 작성
✅ execute(), undo() 구현
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| Undo/Redo 필요 | ⭐⭐⭐ | 히스토리 관리 |
| 작업 큐잉 | ⭐⭐⭐ | 지연 실행 |
| 매크로 | ⭐⭐⭐ | 명령 조합 |
| 로깅/감사 | ⭐⭐⭐ | 추적 가능 |

### 💡 핵심 포인트

1. **요청을 객체로**
2. **디커플링**
3. **Undo 지원**
4. **작업 관리**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Template Method](./03-TemplateMethod.md) | [다음: Iterator →](05-Iterator.md)**

</div>
