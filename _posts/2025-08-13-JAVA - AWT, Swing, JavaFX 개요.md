---
layout: post
title: Java - AWT, Swing, JavaFX 개요
date: 2025-08-13 21:20:23 +0900
category: Java
---
# AWT, Swing, JavaFX 개요

Java에서 **AWT(Abstract Window Toolkit)**, **Swing**, 그리고 **JavaFX**는 GUI(그래픽 사용자 인터페이스) 애플리케이션을 개발하는 대표적인 라이브러리입니다.  
AWT와 Swing은 오래전부터 Java에 포함되어 있었고, JavaFX는 보다 현대적인 UI 프레임워크로 등장했습니다.

---

## 1. AWT (Abstract Window Toolkit)

### 1.1 개요
- **Java 1.0**부터 제공된 최초의 GUI 툴킷
- 운영체제의 네이티브 GUI 컴포넌트를 직접 호출하여 화면을 구성 (**플랫폼 종속적**)
- 버튼, 체크박스, 텍스트 필드 등 기본 컴포넌트 제공
- 이벤트 위임 모델(Event Delegation Model) 사용

### 1.2 특징
- 네이티브 컴포넌트 → 운영체제에 따라 UI 모양과 동작이 달라짐
- 경량이지만 커스터마이징이 어려움
- 하위 레벨 그래픽 처리(`Graphics` 클래스) 지원

### 1.3 예시 코드
```java
import java.awt.*;

public class AWTExample {
    public static void main(String[] args) {
        Frame frame = new Frame("AWT Example");
        Button button = new Button("Click Me");

        button.addActionListener(e -> System.out.println("Button Clicked"));
        frame.add(button);

        frame.setSize(300, 200);
        frame.setLayout(new FlowLayout());
        frame.setVisible(true);
    }
}
```

---

## 2. Swing

### 2.1 개요
- **Java 1.2**부터 제공된 AWT 확장 라이브러리
- AWT 기반이지만 모든 UI 컴포넌트를 **순수 Java로 구현** → 플랫폼 독립적
- **경량 컴포넌트**로 유연한 UI 커스터마이징 가능
- MVC(Model-View-Controller) 패턴 적용

### 2.2 특징
- AWT보다 다양한 컴포넌트(`JButton`, `JTable`, `JTree` 등) 제공
- **Look & Feel** 변경 가능 (Metal, Nimbus, Windows 스타일 등)
- 이벤트 모델은 AWT와 동일

### 2.3 예시 코드
```java
import javax.swing.*;

public class SwingExample {
    public static void main(String[] args) {
        JFrame frame = new JFrame("Swing Example");
        JButton button = new JButton("Click Me");

        button.addActionListener(e -> System.out.println("Button Clicked"));
        frame.add(button);

        frame.setSize(300, 200);
        frame.setLayout(new java.awt.FlowLayout());
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setVisible(true);
    }
}
```

---

## 3. JavaFX

### 3.1 개요
- Java 8 이후 본격 도입된 **최신 GUI 프레임워크**
- CSS 스타일링, FXML(XML 기반 UI 선언), 미디어 처리, 애니메이션 등 **현대적인 UI 기능** 제공
- 2D/3D 그래픽, 오디오·비디오 재생, 웹뷰(WebView) 통합 가능
- **Scene Graph** 구조 사용 → UI 구성 요소를 계층적으로 배치

### 3.2 특징
- **CSS**를 통한 스타일링 → HTML/CSS 개발자 친화적
- **FXML**로 UI와 로직 분리 → MVC/MVVM 아키텍처 구현 용이
- 크로스플랫폼 실행 가능
- 하드웨어 가속 기반 렌더링 지원
- 데스크톱 + 임베디드 환경 모두 지원

### 3.3 예시 코드
```java
import javafx.application.Application;
import javafx.scene.Scene;
import javafx.scene.control.Button;
import javafx.scene.layout.StackPane;
import javafx.stage.Stage;

public class JavaFXExample extends Application {
    @Override
    public void start(Stage primaryStage) {
        Button button = new Button("Click Me");
        button.setOnAction(e -> System.out.println("Button Clicked"));

        StackPane root = new StackPane(button);
        Scene scene = new Scene(root, 300, 200);

        primaryStage.setTitle("JavaFX Example");
        primaryStage.setScene(scene);
        primaryStage.show();
    }

    public static void main(String[] args) {
        launch(args);
    }
}
```

---

## 4. AWT vs Swing vs JavaFX 비교

| 항목 | AWT | Swing | JavaFX |
|------|-----|-------|--------|
| 출시 시기 | Java 1.0 | Java 1.2 | Java 8 이후 |
| 구현 방식 | 네이티브 | 순수 Java | 순수 Java + 하드웨어 가속 |
| 플랫폼 종속성 | 높음 | 낮음 | 낮음 |
| UI 커스터마이징 | 제한적 | 높음 | 매우 높음 (CSS, FXML) |
| 컴포넌트 다양성 | 적음 | 많음 | 매우 많음 + 미디어/그래픽 |
| 2D/3D 그래픽 | 제한적 | 제한적 | 강력한 지원 |
| 추천 용도 | 단순/레거시 | 복잡한 데스크톱 UI | 최신형, 미디어/애니메이션 포함 앱 |

---

## 5. 결론
- **AWT**: 단순하고 빠르지만 커스터마이징과 호환성이 제한적
- **Swing**: 풍부한 컴포넌트와 플랫폼 독립성 제공, 여전히 널리 사용됨
- **JavaFX**: 최신 UI 요구사항, CSS/FXML, 애니메이션, 미디어 처리에 적합  
  → **신규 프로젝트**라면 JavaFX가 더 권장됨