### 日志

Exception in thread "main" java.lang.NoSuchMethodError: 'java.lang.String org.junit.platform.engine.discovery.MethodSelector.getMethodParameterTypes()'
at com.intellij.junit5.JUnit5TestRunnerUtil.loadMethodByReflection(JUnit5TestRunnerUtil.java:127)
at com.intellij.junit5.JUnit5TestRunnerUtil.buildRequest(JUnit5TestRunnerUtil.java:102)
at com.intellij.junit5.JUnit5IdeaTestRunner.startRunnerWithArgs(JUnit5IdeaTestRunner.java:43)
at com.intellij.rt.junit.IdeaTestRunner$Repeater$1.execute(IdeaTestRunner.java:38)
	at com.intellij.rt.execution.junit.TestsRepeater.repeat(TestsRepeater.java:11)
	at com.intellij.rt.junit.IdeaTestRunner$Repeater.startRunnerWithArgs(IdeaTestRunner.java:35)
at com.intellij.rt.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:231)
at com.intellij.rt.junit.JUnitStarter.main(JUnitStarter.java:55)

### 错误原因

spring-boot-starter-test 内部的 junit-jupiter 升级到 6.x 版本，但是 idea 内部未及时更新，仍旧使用 5.x，导致运行时找不到方法

### 解决思路

升级 idea 版本或降级依赖版本，统一到 5.x 版本）

### 采取方案

覆盖处理 spring-boot-starter-test 内部的 junit-jupiter 到 5.x 版本

<dependency>
<groupId>redis.clients</groupId>
<artifactId>jedis</artifactId>
<version>7.1.0</version>
</dependency>

<dependency>
    <groupId>org.junit.platform</groupId>
    <artifactId>junit-platform-commons</artifactId>
    <version>1.14.1</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-api</artifactId>
    <version>5.14.1</version>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-params</artifactId>
    <version>5.14.1</version>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.junit.platform</groupId>
    <artifactId>junit-platform-engine</artifactId>
    <version>1.14.1</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-engine</artifactId>
    <version>5.14.1</version>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.14.1</version>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <version>3.5.7</version>
</dependency>

### 关键：指定的依赖要在被覆盖的依赖之前定义
