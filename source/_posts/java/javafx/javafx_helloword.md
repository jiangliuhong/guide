---
title: 初识JavaFX
categories: Java
date: 2021-01-27 23:30:07
tags:
    - Java
    - JavaFx
cover: https://static.jiangliuhong.top/images/2021/1/1611761730790.png
---

# 初识JavaFX

## 什么是JavaFX

`JavaFX`是`Java`下的图形界面工具包，是一组图形和媒体API，我们可以用它们来进行桌面端程序开发。

`JavaFX`技术有着良好的前景，包括可以直接调用Java API的能力。因为 JavaFX Script是静态类型，它同样具有结构化代码、重用性和封装性，如包、类、继承和单独编译和发布单元，这些特性使得使用JavaFX技术创建和管理大型程序变为可能。

同时基于`Java`的跨平台的特性，`JavaFx`可编译在`windows`、`macos`、`linux`平台下的应用程序。

## 快速构建项目

### 前提条件

安装如下软件：

- java11
- idea
- maven

### 运行helloword

在[openjfx](https://openjfx.cn/openjfx-docs/)网站中,记录中使用`maven`、`gradle`、`idea`等工具构建项目的方法，此处我使用的是`maven`来构建项目，项目`pom.xml`文件内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>top.jiangliuhong.jfx</groupId>
    <artifactId>helloword</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>

        <dependency>
            <groupId>org.openjfx</groupId>
            <artifactId>javafx-controls</artifactId>
            <version>15</version>
        </dependency>
        <dependency>
            <groupId>org.openjfx</groupId>
            <artifactId>javafx-fxml</artifactId>
            <version>15</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>11</source>
                    <target>11</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.openjfx</groupId>
                <artifactId>javafx-maven-plugin</artifactId>
                <version>0.0.5</version>
                <configuration>
                    <mainClass>HelloFX</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

创建程序入口：

```java
import javafx.application.Application;
import javafx.geometry.Pos;
import javafx.scene.Scene;
import javafx.scene.layout.HBox;
import javafx.scene.text.Font;
import javafx.scene.text.FontWeight;
import javafx.scene.text.Text;
import javafx.stage.Stage;

public class Main extends Application {
    public void start(Stage stage) throws Exception {
        Text text = new Text("helloJFX11221");
        text.setFont(Font.font(null, FontWeight.BOLD, 20));
        HBox hBox = new HBox();
        hBox.setPrefWidth(400);
        hBox.setPrefHeight(200);
        hBox.setAlignment(Pos.CENTER);
        hBox.getChildren().add(text);
        Scene scene = new Scene(hBox);
        stage.setScene(scene);
        stage.show();
    }
}
```

直接运行`Main`类，系统会弹出如下窗口，

![](/Users/jiangliuhong/Downloads/1611761458778.png)

### 运行错误

上面的示例在mac上是能够正确运行，但是在windows系统下会提示如下错误：

```
错误: 缺少 JavaFX 运行时组件, 需要使用该组件来运行此应用程序
```

此时需要再增加一个启动类，然后运行即可，代码如下：

```java
import javafx.application.Application;

public class App {

    public static void main(String[] args) {
        Application.launch(Main.class);
    }
}

```

