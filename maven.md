### build

超级pom： 项目中所有pom文件的父类

lib目录下 maven-model-builder-3.8.2.jar文件中  *\org\apache\maven\model\pom-4.0.0.xml* 

**打印有效 pom**

mvn help:effective-pom

##### 定义约定的目录结构

| sourceDirectory       | 主体源程序存放目录         |
| --------------------- | -------------------------- |
| scriptSourceDirectory | 脚本源程序存放目录         |
| testSourceDirectory   | 测试源程序存放目录         |
| outputDirectory       | 主体源程序编译结果输出目录 |
| testOutputDirectory   | 测试源程序编译结果输出目录 |
| resources             | 主体资源文件存放目录       |
| testResources         | 测试资源文件存放目录       |
| directory             | 构建结果输出目录           |



### 依赖排除<exclusions >

```pom
<dependency>
    <groupid>net.javatv.maven</groupid>
    <artifactid>auth</artifactid>
    <version>1.0.0</version>
    <scope>compile</scope>
     
    <!-- 使用excludes标签配置依赖的排除    -->
    <exclusions>
        <!-- 在exclude标签中配置一个具体的排除 -->
        <exclusion>
            <!-- 指定要排除的依赖的坐标（不需要写version） -->
            <groupid>commons-logging</groupid>
            <artifactid>commons-logging</artifactid>
        </exclusion>
    </exclusions>
</dependency>
```

