# Maven安装
## Maven安装细节

bin目录包含了运行Maven的mvn脚本。

boot目录包含一个负责创建maven运行所需要的类装载器的jar文件。

conf目录包含一个全局的settings.xml文件，需要自定义的话，覆写~/.m2目录下的settings.xml文件。

lib目录有一个包含maven核心的jar文件。

### 用户相关配置和仓库
~/.m2/settings.xml 该文件包含用户相关认证，仓库和其他信息配置，自定义maven行为。

~/.m2/repository/ 本地仓库

## 使用maven help插件
help:active-profiles 列出当前构建中活动的profile

help:effective-pom 显示当前构建的实际pom

help:effective-settings 打印项目实际settings

help:describe 描述插件的属性,必须要提供插件的groupId和artifactId，比如:

```
mvn help:describe -Dplugin=help

下面是输出完整的带有参数的目标列表
mvn help:describe -Dplugin=help -Dfull
```

# 一个简单的maven项目
## 创建一个项目
命令行使用Maven Archetype插件

```
mvn archetype:generate -DgroupId=me.cxis.maven.test -DartifactId=maven-test -DpackageName=me.cxis.maven -DarchetypeArtifactId=maven-archetype-webapp
```

archetype:generate通过archetype快速创建一个项目。

## 构建一个项目
构建打包，在包含pom.xml的目录下运行 mvn install

## 项目对象模型（Project Object Model）
pom.xml文件中：

groupId，artifactId，packaging，version是maven的坐标，唯一标识一个项目。

name，url提供描述性元素。

dependencies定义了一个单独的，测试范围依赖。

## 核心概念
### maven插件和目标

### maven生命周期

### maven坐标

### maven仓库

### maven依赖管理

### 站点生成和报告
mvn site

# 定制一个maven项目

## 定制项目信息
licenses，organization，developers

### maven exec插件
exec插件允许运行Java类和其他脚本，用来测试。

## 浏览项目依赖
mvn dependency:resolve

依赖树：mvn dependency:tree

## 执行单元测试
mvn test

### 忽略测试失败

```
<project> 
	[...]	<build> 
		<plugins>			<plugin> 
			<groupId>org.apache.maven.plugins</groupId> 			<artifactId>maven-surefire-plugin</artifactId> 			<configuration>			<testFailureIgnore>true</testFailureIgnore> 
			</configuration>			</plugin> 
		</plugins>	</build>	[...] 
</project>
```

或者

`mvn test -Dmaven.test.failure.ignore=true`

### 跳过单元测试

`mvn install -Dmaven.test.skip=true`

或者

```
<project> 
	[...]	<build> 
		<plugins>			<plugin> 
			<groupId>org.apache.maven.plugins</groupId> 			<artifactId>maven-surefire-plugin</artifactId> 			<configuration>				<skip>true</skip> 
			</configuration>			</plugin> 
		</plugins>	</build>	[...] 
</project>
```

## maven assembly插件
maven assembly插件是一个用来创建应用程序分发包的插件。

配置maven的装配描述符

```
<project> 
	[...]	<build> 
		<plugins>			<plugin> 
				<groupId>maven-assembly-plugin</groupId>						<configuration>							<descriptorRefs> 								<descriptorRef>jar-with-dependencies</descriptorRef>							</descriptorRefs>
						</configuration>			</plugin> 
		</plugins>	</build>	[...] 
</project>
```

使用 `mvn install assembly:assembly`

# 一个简单的web应用
packaging元素的值是war。

## 配置jetty插件

```
<project>
	[...]		<build>
			<finalName>simple-webapp</finalName>
			<plugins>				<plugin>
					<groupId>org.mortbay.jetty</groupId> 					<artifactId>maven-jetty-plugin</artifactId>				</plugin>
			</plugins>		</build>	[...] 
</project>
```

使用 `mvn jetty:run`

编译项目：`mvn compile`

Maven Dependency插件能够帮助你发现对于依赖的直接引用`mvn dependency:analyze`

# 项目对象模型

## pom
pom包含四类描述和配置

1. 项目总体信息 包含项目的名称，url，组织，开发者，许可证。
2. 构建设置 自定义maven构建的默认行为。
3. 构建环境 包含不同环境中使用的profile。
4. pom关系 依赖等。

### 超级pom
定义了一组被所有项目共享的默认设置

1. 定义了单独的远程maven仓库，id为central，可通过settings.xml来覆盖。
2. 中央maven仓库同时也包含maven插件
3. build元素设置maven标准目录布局中的默认值
4. 超级pom为核心插件提供了默认版本

### 有效pom
查看有效pom：`mvn help:effective-pom`

## pom语法

### 项目版本

###属性引用
pom可以使用`${}`包含对属性的引用。

maven读取pom的时候，会在载入pom的时候替换这些属性的引用。

* 三个隐式变量：

env 操作系统或shell的环境变量

project 暴露pom，可以使用`${project.artifactId}`

settings 暴露settings信息，如`${settings.offline}`，会引用settings.xml文件中的offline的元素的值。

* 还可以引用系统属性，以及任何在pom中和构建profile中自定义的属性组

java系统属性 所有可通过java.lang.System中getProperties()方法访问的属性都被暴露成pom属性。

我们还可以通过pom.xml或settings.xml中的properties元素设置自己的属性。

## 项目依赖

### 依赖范围

* compile 是默认的范围，依赖在所有的classpath中可用，它们会被打包。
* provided 只有在当jdk或者容器已经提供该依赖之后才使用。在编译classpath可用，不会被打包。
* runtime 在运行和测试系统的时候需要，编译的时候不需要。
* test 编译和运行都不需要，只在测试编译和测试运行阶段可用。
* system 与provided类似，但是必须显式的提供一个对于本地系统中的jar文件路径。

### 可选依赖
添加`<optional>true<optional>`

### 依赖版本界限
`(1,4)` 大于1小于4

`[1,4]` 大于等于1小于等于4

`<version>[3.8,4.0)</version>`大于等于3.8，小于4.0

`<version>[,3.8.1]</version>` 小于3.8.1

### 冲突解决
需要排除一个依赖：

```
<dependency>
	<groupId>xxx</groupId>
	<artifactId>sss</artifactId>
	<version>1.0</version>	<exclusions>		<exclusion>
			<groupId>www</groupId>
			<artifactId>asd</artifactId>		</exclusion>
	</exclusions></dependency>
```

### 依赖管理
dependencyMangement

# 构建生命周期
有三种标准的生命周期，清理clean，默认default，站点site

### 清理 clean
`mvn clean`包含了三个生命周期阶段

* pre-clean
* clean
* post-clean

目标clean:clean通过删除构建目录删除整个构建的输出。

### 默认生命周期 default
是一个软件应用程序构建过程的总体模型，第一个阶段是validate，最后一个阶段是deploy。

### 站点生命周期
生成项目文档和报告，包含四个阶段：

1. pre-site
2. site
3. post-site
4. site-deploy

默认绑定到站点生命周期的目标：
` site - site:site`
` site-deploy - site:deploy`

生成一个站点 `mvn site`

## 打包相关生命周期