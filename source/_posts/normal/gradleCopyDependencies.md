---
title: gradle copy dependencies
categories: 
    - Gradle
date: 2021-04-30 10:11:00
tags:
    - tools
---

> 在对项目进行打包的时候，可能会遇见这样的需求，那就是将当前项目依赖的jar包，复制到某个路径，例如`lib`目录，在maven体系中，需要使用对应的插件，而在`gradle`则只需要编写对应的脚本即可。

## 在maven中的方式

在maven中，只需要使用对应的build插件即可，插件代码如下：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>3.1.1</version>
    <executions>
        <!-- package 阶段将项目的依赖包拷贝到项目的 lib 目录下 -->
        <execution>
            <id>copy-dependencies</id>
            <phase>package</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <outputDirectory>${project.basedir}/lib</outputDirectory>
                <includeScope>runtime</includeScope>
            </configuration>
        </execution>
    </executions>
</plugin>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-antrun-plugin</artifactId>
    <version>1.8</version>
    <executions>
        <!-- prepare-package 阶段将项目下的 lib 目录删除，以便下次重新生成新的 lib 目录 -->
        <execution>
            <id>clean</id>
            <phase>prepare-package</phase>
            <configuration>
                <target>
                    <delete quiet="true">
                        <fileset dir="${project.basedir}/lib" includes="*.*"/>
                    </delete>
                </target>
            </configuration>
            <goals>
                <goal>run</goal>
            </goals>
        </execution>
        <!-- package 阶段将项目自身的 jar 产出拷贝到项目的 lib 目录下 -->
        <execution>
            <id>copy</id>
            <phase>package</phase>
            <configuration>
                <target>
                    <copy file="${project.build.directory}/${project.artifactId}-${project.version}.jar"
                          tofile="${project.basedir}/lib/${project.artifactId}-${project.version}.jar"/>
                </target>
            </configuration>
            <goals>
                <goal>run</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

两个插件的作用分别将项目依赖的包以及自己本身的jar包拷贝到lib目录中。

## 在`gradle`中的方式

在`gradle`中，我们只需要编写对应的任务，然后将其放在build任务执行之后执行即可，首先看下具体脚本：

```groovy
configurations {
    customConfig.extendsFrom api
}

task copyLib(type: Copy) {
    // 先删除lib下的jar包
    def libPath = projectDir.absolutePath + '/lib'
    println "${project.name} do copyLib ： ${libPath}"
    def libDir = file(libPath)
    if (libDir.exists()) {
        libDir.listFiles().findAll {
            it.name.endsWith('.jar')
        }.collect {
            delete it
        }
    }
    dependsOn jar
    String thisJarName = "${archivesBaseName}-${version}.jar"
    String thisJarPath = "${getBuildDir()}/libs/${thisJarName}"
    // 拷贝自身
    from(file(thisJarPath)) into file(libPath)
    // 拷贝依赖包
    from(project.configurations.customConfig) into(libPath)
}
// 每次构建之后，执行下lib的拷贝
build.finalizedBy(copyLib)
```

**注意：**

- 这里我使用了`api`去设置依赖，使用`api`需要在`build.gradle`中引入插件：`apply plugin: 'java-library'`
- 在`configurations`中定义了`customConfig`,这是由于如果在`from`中直接使用`configurations.api`，则会报如下错误。

```
Resolving dependency configuration 'api' is not allowed as it is defined as 'canBeResolved=false'.
  Instead, a resolvable ('canBeResolved=true') dependency configuration that extends 'api' should be resolved.
```

​	同时也可能会报这样的错误

```
 Could not get unknown property 'api' for configuration container of type org.gradle.api.internal.artifacts.configurations.DefaultConfigurationContainer.
```

  参考网络上的，说推荐使用`configurations.implementation`或者`configurations.compile`，但可能由于`gradle`版本(这里我使用的是`gradle 7.0`)的不同，以及项目构建的不同，都会报`Could not get unknown property`的错误。推荐还是在`configurations`中定义一个属性，这里我是定义了一个`customConfig`并且他继承了`api`，所以能够将依赖拷贝到lib目录下