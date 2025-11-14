---
layout: post
title: Java - AWT, Swing, JavaFX 개요
date: 2025-08-13 21:20:23 +0900
category: Java
---
# AWT, Swing, JavaFX

## 공통 큰 그림과 역사

| 항목 | AWT | Swing | JavaFX (OpenJFX) |
|---|---|---|---|
| 초기 도입 | JDK 1.0 | JDK 1.2 | Java 8(동봉) → **JDK 11부터 분리(OpenJFX)** |
| 구현 철학 | **네이티브 컴포넌트(heavyweight)** 래핑 | **경량(lightweight)** 순수 Java 위젯 | **Scene Graph + 하드웨어 가속** |
| 스레드 모델 | AWT Event Dispatch Thread (EDT) | **EDT** (AWT와 동일) | **JavaFX Application Thread** |
| 스타일링 | OS 룩앤필 | 룩앤필(L&F) 교체 | **CSS** 기반, **FXML**로 선언적 UI |
| 그래픽 | java.awt.Graphics | Swing + **Graphics2D**(안티에일리어싱 등) | **GPU 가속**, 2D/3D, 애니메이션/미디어 |
| 배포 | JAR/네이티브 래퍼 | 동일 | **jpackage**, **jlink**로 경량 런타임 |
| 권장 용도 | 레거시/단순 | 풍부한 데스크톱 UI(레거시 포함) | **신규/현대적 UI**, 미디어·애니메이션 |

> **현업 권장**: 신규 프로젝트는 **JavaFX**를 우선 검토. 기존 Swing/AWT 자산이 클 경우 점진적 유지 또는 **Swing ↔ JavaFX 상호운용**으로 혼합 전환.

---

## AWT (Abstract Window Toolkit)

### 핵심 요약

- OS 네이티브 위젯을 감싼 **heavyweight** 컴포넌트.
- **이벤트 위임 모델**(delegation model)을 최초 도입.
- 저수준 드로잉 `Graphics`, 입력 디바이스, 클립보드 등 **플랫폼 브리지** 역할.

### 최소 예제

```java
import java.awt.*;
import java.awt.event.*;

public class AWTExample {
    public static void main(String[] args) {
        Frame frame = new Frame("AWT Example");
        Button button = new Button("Click Me");
        button.addActionListener(e -> System.out.println("Clicked"));
        frame.add(button);
        frame.setLayout(new FlowLayout());
        frame.setSize(320, 200);
        frame.addWindowListener(new WindowAdapter() {
            public void windowClosing(WindowEvent e) { frame.dispose(); }
        });
        frame.setVisible(true);
    }
}
```

### 장단점 & 실전 팁

- 장점: **네이티브 룩**, OS 자원 접근.
- 단점: 플랫폼별 차이, **커스터마이징 제약**, heavyweight 혼합 시 Z-order 문제.
- 팁: 레거시 유지/OS 통합(프린팅/클립보드) 시에만 선택. 신규는 Swing/JavaFX 권장.

---

## Swing

### 핵심 요약

- **경량 컴포넌트**: 그리기를 Java가 담당 → 일관된 UI·고급 커스터마이징.
- **EDT(Event Dispatch Thread)** 에서 모든 UI 변경 수행.
- MVC 설계(모델 분리), **룩앤필(L&F)** 변경 가능(Nimbus 등).

### EDT 규칙 (매우 중요)

- **UI 생성/갱신은 항상 EDT에서**.
- `SwingUtilities.invokeLater(...)` 또는 `invokeAndWait(...)` 사용.

```java
import javax.swing.*;

public class SwingHello {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            JFrame f = new JFrame("Swing");
            JButton b = new JButton("Click");
            b.addActionListener(e -> System.out.println("Click"));
            f.add(b);
            f.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
            f.setSize(320, 200);
            f.setLocationRelativeTo(null);
            f.setVisible(true);
        });
    }
}
```

### 레이아웃 매니저 요약

| 매니저 | 특징 | 사용 예 |
|---|---|---|
| `BorderLayout` | 북/남/동/서/중앙 | 메인 프레임 기본 |
| `FlowLayout` | 좌→우 흐름 | 툴바/버튼 나열 |
| `GridLayout` | 균등 격자 | 동일 크기 버튼 |
| `BoxLayout` | 수직/수평 스택 | 폼/패널 정렬 |
| `GridBagLayout` | **유연**하나 복잡 | 복합 폼 |

```java
JPanel p = new JPanel(new BorderLayout());
p.add(new JButton("North"), BorderLayout.NORTH);
p.add(new JButton("Center"), BorderLayout.CENTER);
```

### 커스텀 페인팅 (Graphics2D)

Swing에서는 `JComponent#paintComponent`를 오버라이드한다.

```java
import javax.swing.*;
import java.awt.*;

class Gauge extends JComponent {
    @Override protected void paintComponent(Graphics g) {
        super.paintComponent(g);
        var g2 = (Graphics2D) g.create();
        try {
            g2.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
            int w = getWidth(), h = getHeight();
            g2.setColor(new Color(30,144,255));
            g2.fillRoundRect(10, h/2 - 15, w - 20, 30, 20, 20);
        } finally { g2.dispose(); }
    }
}

public class PaintDemo {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            JFrame f = new JFrame("Paint");
            f.add(new Gauge());
            f.setSize(400, 200);
            f.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
            f.setVisible(true);
        });
    }
}
```

### 백그라운드 작업 (SwingWorker)

```java
new SwingWorker<String, Void>() {
    @Override protected String doInBackground() throws Exception {
        Thread.sleep(500); return "done";
    }
    @Override protected void done() {
        try { System.out.println(get()); } catch (Exception ignored) {}
    }
}.execute();
```

### 룩앤필 & 하이 DPI

```java
UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName());
```
- 최신 JDK는 HiDPI 자동 스케일링 지원. 사용자 OS 스케일링을 존중.

---

## JavaFX (OpenJFX)

### 핵심 요약

- **Scene Graph**(트리)로 UI 구성, **GPU 가속** 렌더링.
- **JavaFX Application Thread**에서 UI 변경.
- **CSS 스타일링**, **FXML**로 선언적 UI, **프로퍼티/바인딩**으로 상태 동기화.
- 2D/3D, **애니메이션**, **Media**, **WebView**까지 포괄.

### 최소 예제

```java
import javafx.application.Application;
import javafx.scene.Scene;
import javafx.scene.control.Button;
import javafx.scene.layout.StackPane;
import javafx.stage.Stage;

public class FxHello extends Application {
    @Override public void start(Stage stage) {
        Button btn = new Button("Click");
        btn.setOnAction(e -> System.out.println("Click"));
        stage.setScene(new Scene(new StackPane(btn), 320, 200));
        stage.setTitle("JavaFX");
        stage.show();
    }
    public static void main(String[] args) { launch(args); }
}
```

### 레이아웃 & 컨트롤

| 레이아웃 | 용도 |
|---|---|
| `HBox`/`VBox` | 수평/수직 스택 |
| `BorderPane` | 상/하/좌/우/중앙 |
| `GridPane` | 폼/격자 |
| `StackPane` | 겹치기 배치 |
| `AnchorPane` | 절대 위치 고정 |

```java
var root = new javafx.scene.layout.BorderPane();
root.setTop(new javafx.scene.control.MenuBar());
root.setCenter(new javafx.scene.control.TableView<>());
```

### CSS 스타일링

- `-fx-` 접두사의 CSS 속성.

```css
/* app.css */
.root { -fx-font-size: 14px; }
.button.primary { -fx-background-color: #1e90ff; -fx-text-fill: white; }
```

```java
Scene scene = new Scene(root, 360, 240);
scene.getStylesheets().add(getClass().getResource("/app.css").toExternalForm());
Button b = new Button("Save"); b.getStyleClass().add("primary");
```

### FXML + 컨트롤러 (선언적 UI)

`sample.fxml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<?import javafx.scene.control.*?>
<?import javafx.scene.layout.*?>
<BorderPane fx:controller="demo.Controller" xmlns:fx="http://javafx.com/fxml">
  <top><ToolBar><Button text="Run" onAction="#run"/></ToolBar></top>
  <center><TextArea fx:id="area"/></center>
</BorderPane>
```

`Controller.java`
```java
package demo;
import javafx.fxml.FXML;
import javafx.scene.control.TextArea;

public class Controller {
    @FXML private TextArea area;
    @FXML public void run() { area.appendText("running...\n"); }
}
```

로딩:
```java
var loader = new javafx.fxml.FXMLLoader(getClass().getResource("/sample.fxml"));
Scene scene = new Scene(loader.load());
stage.setScene(scene);
```

### 프로퍼티와 바인딩

```java
import javafx.beans.property.*;

StringProperty name = new SimpleStringProperty("Alice");
javafx.scene.control.Label label = new javafx.scene.control.Label();
label.textProperty().bind(name.concat(" !")); // name 변경 시 자동 반영
name.set("Bob"); // Label 텍스트: "Bob !"
```

### 애니메이션 & 미디어

```java
import javafx.animation.*;
import javafx.util.Duration;
import javafx.scene.shape.Rectangle;

Rectangle r = new Rectangle(20,20,50,50);
TranslateTransition tt = new TranslateTransition(Duration.millis(600), r);
tt.setToX(200); tt.setAutoReverse(true); tt.setCycleCount(Animation.INDEFINITE);
tt.play();
```

### WebView 임베드

```java
javafx.scene.web.WebView web = new javafx.scene.web.WebView();
web.getEngine().load("https://openjdk.org/");
```

### 백그라운드 작업 (Task/Service)

```java
import javafx.concurrent.Task;

Task<String> task = new Task<>() {
    @Override protected String call() throws Exception {
        Thread.sleep(500); return "done";
    }
};
task.setOnSucceeded(e -> System.out.println(task.getValue()));
new Thread(task, "worker").start();
```

---

## Swing ↔ JavaFX 상호 운용

| 방향 | 컴포넌트 | 설명 |
|---|---|---|
| Swing 안에 JavaFX | **`JFXPanel`** | 기존 Swing 앱에 JavaFX 뷰 삽입 |
| JavaFX 안에 Swing | **`SwingNode`** | JavaFX Scene에 Swing 컴포넌트 삽입 |

### Swing → JavaFX (JFXPanel)

```java
import javafx.embed.swing.JFXPanel; // JavaFX 스레드 초기화 트리거
import javax.swing.*;

public class FxInSwing {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            JFrame f = new JFrame("JFX in Swing");
            JFXPanel panel = new JFXPanel(); // 생성 시 JavaFX runtime 준비
            f.add(panel);
            f.setSize(400,300);
            f.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
            f.setVisible(true);

            // JavaFX 스레드에서 Scene 설정
            javafx.application.Platform.runLater(() -> {
                var btn = new javafx.scene.control.Button("FX");
                panel.setScene(new javafx.scene.Scene(new javafx.scene.layout.StackPane(btn)));
            });
        });
    }
}
```

> **스레드 교차 호출 주의**: Swing은 EDT, JavaFX는 Application Thread. 각각의 **runLater**를 준수.

---

## 프로젝트 설정 (Maven/Gradle) & 모듈(JPMS)

### JavaFX 의존성 (JDK 11+ OpenJFX 분리)

**Gradle(Kotlin DSL)**:
```kotlin
plugins {
    application
    id("org.openjfx.javafxplugin") version "0.0.14"
}
javafx {
    version = "21"
    modules = listOf("javafx.controls","javafx.fxml","javafx.web")
}
application {
    mainClass.set("demo.FxHello")
}
```

**Maven**:
```xml
<dependencies>
  <dependency>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-controls</artifactId>
    <version>21</version>
  </dependency>
  <dependency>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-fxml</artifactId>
    <version>21</version>
  </dependency>
</dependencies>
```

> 일부 환경에서는 플랫폼 분류자(`:win`, `:mac`, `:linux`)가 필요한 배포 구성이 있으며, Gradle 플러그인이 이를 간소화한다.

### `module-info.java` (JPMS)

```java
module demo.app {
    requires javafx.controls;
    requires javafx.fxml;
    opens demo to javafx.fxml;          // FXML 리플렉션
    exports demo;
}
```

---

## 배포: jlink & jpackage

### jlink — 경량 런타임 이미지

- **필요 모듈만 포함한 JRE**를 생성하여 배포 크기↓, 시작 속도↑.

```bash
jlink --module-path $JAVA_HOME/jmods:mods \
      --add-modules demo.app,javafx.controls,javafx.fxml \
      --strip-debug --compress=2 --no-header-files --no-man-pages \
      --output build/image
```

### jpackage — 네이티브 설치본

- `.msi/.pkg/.dmg/.deb/.rpm` 등 **OS 네이티브 설치 파일** 생성.

```bash
jpackage --name DemoApp \
         --input build/libs \
         --main-jar demo-app.jar \
         --main-class demo.FxHello \
         --type app-image
```

> CI에 통합해 **재현 가능한 배포 아티팩트**를 만든다. OpenJFX 네이티브 포함 시 플랫폼별 파이프라인 분기 필요.

---

## 고급 주제

### AWT/Swing heavyweight vs lightweight 혼합

- Swing은 lightweight, AWT 일부는 heavyweight. 과거 **혼합 시 Z-order/포커스** 문제가 있었다.
- 최신 JDK는 대부분 개선되었지만, **비디오/브라우저 임베드** 등 일부 케이스에서 주의.

### 페인팅 파이프라인 차이

- **Swing**: OS 윈도우→Java 더블버퍼→`paintComponent` 호출 흐름. `RepaintManager`가 스케줄링.
- **JavaFX**: Scene Graph 트리를 **펄스(pulse)** 주기로 평가하고 GPU로 플러시. 레이아웃→CSS→렌더 단계 파이프라인.

### 접근성/국제화

- Swing/JavaFX 모두 접근성 API, 입력기(IM) 및 **RTL 레이아웃** 일부 지원.
- i18n: `ResourceBundle`로 문자열 분리, FXML에서 `resources` 속성으로 주입 가능.

### 테스트 전략

- Swing: **AssertJ-Swing**, **Jemmy** 등 UI 테스트 프레임워크.
- JavaFX: **TestFX**, headless 모드 설정(서버 CI).

---

## 성능·메모리·스레딩 체크리스트

1. **EDT/JFX thread 규칙**: 모든 UI 변경은 전용 스레드에서.
   - Swing: `SwingUtilities.invokeLater`
   - JavaFX: `Platform.runLater`
2. **백그라운드 작업 분리**: I/O·CPU 바운드는 `SwingWorker` / `Task` 사용.
3. **바인딩/리스너 누수 방지**: 제거/해제(`removeListener`, `unbind`)와 생명주기 관리.
4. **이미지 캐시/리소스 관리**: 큰 이미지 로딩은 백그라운드 + 약참조/캐시 정책.
5. **레이아웃 최소화**: 빈번한 re-layout 유발 회피(노드 대량 추가는 배치 후 일괄 add).
6. **페인팅 최적화**: Swing은 `paintComponent`에서만 커스텀 드로잉, JavaFX는 캔버스/노드 적절 선택.

---

## 실전 UI 조각 레시피

### Swing TableModel (대용량 데이터 가상화)

```java
import javax.swing.table.AbstractTableModel;

class LazyModel extends AbstractTableModel {
    private final int rows = 1_000_000;
    public int getRowCount() { return rows; }
    public int getColumnCount() { return 3; }
    public Object getValueAt(int r, int c) { return "R" + r + "C" + c; }
}
```

### JavaFX TableView + ObservableList

```java
import javafx.collections.*;
import javafx.scene.control.*;

ObservableList<Person> items = FXCollections.observableArrayList();
TableView<Person> table = new TableView<>(items);
TableColumn<Person,String> name = new TableColumn<>("Name");
name.setCellValueFactory(c -> c.getValue().nameProperty());
table.getColumns().add(name);
```

### JavaFX 3D 간단 샘플

```java
import javafx.scene.*;
import javafx.scene.paint.Color;
import javafx.scene.shape.Box;
import javafx.scene.transform.Rotate;

Box box = new Box(100,100,100);
box.setRotationAxis(Rotate.Y_AXIS);
box.setRotate(30);
Group g = new Group(box);
Scene scene = new Scene(g, 600, 400, true); // depthBuffer=true
scene.setFill(Color.BLACK);
```

---

## AWT vs Swing vs JavaFX — 확장 비교표

| 비교축 | AWT | Swing | JavaFX |
|---|---|---|---|
| 위젯 구현 | 네이티브 | 순수 Java | 순수 Java + GPU |
| 스레드 | AWT-EDT | **EDT** | **FX App Thread** |
| 선언적 UI | 없음 | 없음 | **FXML** |
| 스타일링 | OS 테마 | L&F | **CSS** |
| 애니메이션 | 수동 타이머 | 수동 타이머 | **Transition/Timeline** |
| 미디어/웹 | 제한적 | 별도 라이브러리 | **Media/WebView** |
| 2D/3D | 기본 2D | 2D(강화) | **2D/3D** |
| 데이터 바인딩 | 수동 | 수동 | **Property/Binding** |
| 상호운용 | - | JFXPanel/SwingNode | JFXPanel/SwingNode |
| 배포 | JAR | JAR | **jlink/jpackage** 유리 |

---

## 자주 묻는 질문(FAQ)

**Q1. Swing은 deprecated 인가?**
A. **아니다.** 여전히 지원되며 대규모 레거시 현장에 많다. 다만 **신규**는 JavaFX 권장.

**Q2. JavaFX는 JDK에 없나?**
A. **JDK 11부터 분리(OpenJFX)**. Maven/Gradle 의존성 추가가 필요하다.

**Q3. 고해상도(HiDPI) 문제?**
A. 최신 LTS에서 자동 스케일링이 개선됨. 커스텀 드로잉 시 벡터/비율 기준으로 연산하라.

**Q4. 네이티브 설치본 만들기?**
A. **jpackage** 사용. 플랫폼별 실행 파일/인스톨러 생성.

---

## 마무리 가이드

- **신규 개발**: JavaFX(Controls/FXML/CSS/Animation/WebView/Media) + **jpackage**.
- **기존 Swing 대규모 자산**: 유지/개선 + 부분 화면 JavaFX 삽입(**JFXPanel**)로 점진 전환.
- **공통 원칙**: 전용 UI 스레드 준수(EDT/Fx Thread), 백그라운드 작업 분리, 리소스/리스너 수명주기 관리.
- **배포**: jlink로 경량 런타임 → jpackage로 OS 네이티브 아티팩트.

---

## 부록 — 빌드 스니펫 모음

### Gradle(JavaFX + 애플리케이션)

```kotlin
plugins {
    application
    id("org.openjfx.javafxplugin") version "0.0.14"
}
repositories { mavenCentral() }
javafx {
    version = "21"
    modules = listOf("javafx.controls","javafx.fxml","javafx.web")
}
application { mainClass.set("demo.FxHello") }
```

### Maven(JavaFX)

```xml
<dependencies>
  <dependency>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-controls</artifactId>
    <version>21</version>
  </dependency>
  <dependency>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-fxml</artifactId>
    <version>21</version>
  </dependency>
</dependencies>
```

### module-info.java

```java
module demo.app {
    requires javafx.controls;
    requires javafx.fxml;
    opens demo to javafx.fxml;
    exports demo;
}
```

---

## 참고 체크리스트

- [ ] UI 스레드(EDT/FX) 이외에서 UI 변경 금지
- [ ] 백그라운드 작업(네트워크/IO/CPU)은 SwingWorker/Task로
- [ ] 레이아웃/페인팅/바인딩 성능 점검(프로파일링)
- [ ] 리스너/바인딩 해제와 메모리 누수 점검
- [ ] jlink/jpackage 기반 배포 파이프라인 자동화

---

### 끝.

본 문서는 구식 정보를 제거하고, **JDK 11 이후 OpenJFX 분리**, **jlink/jpackage** 등 최신 배포 스택을 반영했다. 필요 시 이 글의 코드 조각을 바로 복사해 실습할 수 있다.
