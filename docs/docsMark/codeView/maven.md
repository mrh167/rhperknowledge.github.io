### **使用MAVEN遇到的问题**
一、手动添加的jar在idea中加载无法扫描到该jar包
    
场景描述
有一个jar包无法通过maven自动识别下载，即maven的远程仓库里没有该jar包（或者有依赖坐标，但是下载不了）如 aspose-words
maven地址：![1](D:\soft\tools\codeTools\gitFIles\rhperknowledge.github.io\docs\docsMark\codeView\assert\maven\1.png)



![2](D:\soft\tools\codeTools\gitFIles\rhperknowledge.github.io\docs\docsMark\codeView\assert\maven\2.png)



这个时候需要在网上找到该jar包，并将jar包保存到本地仓库里，再让项目导入定位该jar包，项目确实可以运行了

但是，如果将项目打成jar包时， aspose-words并不会打进包，即这个项目还是有问题的。
手动导入jar包maven是无法识别到，因为maven在自动下载jar包时还会生成几个其他文件，因为手动导jar包没有这几个文件，导致识别失败。

![3](D:\soft\tools\codeTools\gitFIles\rhperknowledge.github.io\docs\docsMark\codeView\assert\maven\3.png)



## 二、解决办法

使用mvn的命令行生成所需文件使maven识别

### 步骤

1、带生效pom文件坐标（可以参照maven仓库在结合已下载的jar包版本修改）
我在网上找到15.8.0的包，所以我改了一下版本

```java
<!-- https://mvnrepository.com/artifact/com.aspose/aspose-words -->
<dependency>
    <groupId>com.aspose</groupId>
    <artifactId>aspose-words</artifactId>
    <version>15.8.0</version>
</dependency>
```

2、手动下载的jar包，存放在一位置；

![4](D:\soft\tools\codeTools\gitFIles\rhperknowledge.github.io\docs\docsMark\codeView\assert\maven\4.png)

3、查看MAVEN_HOME配置信息，查看默认使用的maven位置，再查看对应的setting.xml里依赖下载的位置（不要跟项目中使用的maven配置混淆了，可能有些同学电脑里不止一个maven，不止一个setting.xml文件，不止一个依赖存放的位置，不要弄错了，这一步就是为了知道在执行命令行后能找到生成文件的位置，很多同学因为找错位置以为生成不成功）
![5](D:\soft\tools\codeTools\gitFIles\rhperknowledge.github.io\docs\docsMark\codeView\assert\maven\5.png)

4、执行mvn命令（如果配置MAVEN_HOME可以在任意位置打开cmd执行）

**mvn命令格式：**

```java
mvn install:install-file -Dfile=jar包地址（最好不要出现中文路径） -DgroupId=<groupId>标签内的内容 -DartifactId=<artifactId>标签内的内容  -Dversion=<version>标签内的内容 -Dpackaging=jar
```

格式填写注释：

- Dfile：填写手动下载的jar包放置的位置，位置任意，只要能正确指向jar包即可（运行完命令后该jar包可以删除）
- DgroupId：对应待生效坐标< groupId > 的值
- DartifactId：< artifactId > 的值
- Dversion： < version > 的值
- 我下载的jar包在D盘根目录，即对应的命令为：

```java
mvn install:install-file -Dfile=D:\aspose-words-15.8.0-jdk16.jar -DgroupId=com.aspose -DartifactId=aspose-words -Dversion=15.8.0 -Dpackaging=jar
```

执行之后，会在本地仓库下载jar包及maven指向生效文件，这时该坐标就可以使用了

![6](D:\soft\tools\codeTools\gitFIles\rhperknowledge.github.io\docs\docsMark\codeView\assert\maven\6.png)

note：

1、可能下载失败，跟顺序有关，Dfile信息需要放在最前面

2、可能会找不到下载的文件，是因为MAVEN_HOME使用的setting文件没有配置到自己指定的依赖库路径，所以下载到了默认的依赖库里了，可以通过

参考文章：
https://www.cnblogs.com/xm970829/p/13434428.html
jar包下载地址：https://www.cnblogs.com/qiwu1314/p/6101400.html





