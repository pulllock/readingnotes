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

### jar
jar 是默认的打包类型。

process-resources resources:resources

complie complier:complie

process-test-resources resources:testResources

test-compile compiler:testCompile

test surefire:test

package jar:jar

install install:install

deploy deploy:deploy

### pom
pom是最简单的打包类型

package site:attach-descriptor

install install:install

deploy deploy:deploy

### maven plugin
和Jar类似

generate-resources plugin:descriptor

process-resources resources:resources

compile compiler:compile

process-test-resources resources:testResources

test-compile compiler:testCompile

test surefire:test

package jar:jar,plugin:addPluginArtifactMetadata

install install:install,plugin:updateRegistry

deploy deploy:deploy

### ejb

和jar类似 只不过package阶段如下：

package ejb:ejb

### war
和jar类似

package war:war

### ear

generate-resources ear:generate-application-xml       
process-resources resources:resourcespackage ear:earinstall install:installdeploy deploy:deploy

## 通用生命周期

### process resources
resources:resources 目标绑定到process-resources阶段，处理资源并将资源复制到输出目录。

同时也会在资源上应用过滤器，替换资源中的一些符号。

### Compile
compile:compile会编译src/main/java中的所有内容至target/calsses。Compiler插件调用javac。

可在pom中设置版本

```
<project>
	...	<build>
		...		<plugins>
			<plugin>				<artifactId>maven-compiler-plugin</artifactId> 
				<configuration>					<source>1.5</source>					<target>1.5</target>
				</configuration>			</plugin>
		</plugins> 
		...	</build>	... 
</project>
```
配置的是Compiler插件，不是compile:compile目标。

要为compile:compile目标配置，需要放到compile:compile目标的execution元素下。

定义源码的位置：

```
<build>
	...
	<sourceDirectory>src/java</sourceDirectory>
	<outputDirectory>classes</outputDirectory>
	...
</build>
```

### process test resources
和process-resources差不多

### test compile
基本和compile差不多

### Test
Surefire插件是maven的单元测试插件。

默认会寻找测试源码目录下所有以*Test结尾的类。

运行mvn test 之后，在target/surefire-reports目录生成了许多报告。

单元测试失败的时候，默认是停止构建，可以使用testFailureIgnore配置为true，跳过失败。

```
<build>
	<plugins>		<plugin>
			<groupId>org.apache.maven.plugins</groupId> 		<artifactId>maven-surefire-plugin</artifactId>		<configuration>
				<testFailureIgnore>true</testFailureIgnore>		</configuration>
		</plugin>		...	</plugins> 
</build>
```

跳过整个测试：

`mvn install -Dmaven.test.skip=true`

maven.test.skip变量同时控制Complier和Surefire插件。

### install
install:install目标是将项目的主要构件安装到本地仓库。

### deploy
该阶段用来将一个构件部署到远程maven仓库。

部署设置通常在settings.xml中。

# 构建Profile
## 通过maven profiles 实现可移植性
profile是一组可选的配置，用来设置或覆盖配置默认值。可为不同环境定制构建。

profile在pom.xml中配置，给定一个id，运行maven的时候使用命令告诉maven运行特定的profile中的目标。

```
<profiles>	<profile>
		<id>production</id>
		<build>
			<plugins>
				<plugin>					<groupId>org.apache.maven.plugins</groupId> 			<artifactId>maven-compiler-plugin</artifactId> 		<configuration>						<debug>false</debug>#						<optimize>true</optimize>
					</configuration>				</plugin>
			</plugins>		</build>
	</profile></profiles>
```

profiles 通常是pom.xml中的最后一个元素。

必须要有一个id元素，使用的时候命令用`-P<profile_id>`调用。

profile下可以包含其他的元素，上面的代码覆盖了Compiler插件的行为

使用：`mvn clean install -Dproduction -x`

-X 调试。

### 覆盖一个项目对象模型
profile可以覆盖几乎所有的pom.xml中的配置。

## 激活profile

```
<profiles>	<profile>
		<id>jdk16</id>
		<activation>
			<jdk>1.6</jdk>
		</activation>
		<modules>
			<module>simple-script</module> 
		</modules>	</profile>
</profiles>
```
如上配置，在jdk1.6下运行mvn install 可以构建子目录simple-script，在jdk1.5下运行就不会构建simple-script子模块。

### 激活配置
激活配置元素下可以包含jdk版本，操作系统参数，文件，属性等。

### 通过属性缺失激活
可以基于属性如environment.type的值来激活profile。

## 外部profile
可以单独建立一个profiles.xml文件。

## Settings Profile
可以在settings.xml中配置profile

~/.m2/settings.xml 用户配置

maven/conf/settings.xml	全局配置

## 列出活动的profile
`mvn help:active-profiles` 列出所有激活的profile以及他们在哪里定义。

# Maven套件
Assembly

命令行使用assembly:assembly

绑定生命周期使用single

### 预定义的套件描述符

* bin 
* jar-with-dependencies
* project
* src

### 构建一个套件
Assembly可以以两种方式运行：直接命令行调用或者绑定到声明周期阶段将其配置成标准构建过程的一部分。

将项目复制一份

`mvn -DdescriptorId=project assemble:single`

生成可运行jar，如果没有任何依赖，使用jar插件的archive配置就能做到。有依赖的话使用以下配置生成可运行jar：

```
<build>
	<plugins>		<plugin>
			<artifactId>maven-assembly-plugin</artifactId> 			<version>2.2-beta-2</version>			<executions>				<execution>
					<id>create-executable-jar</id> 					<phase>package</phase>
					<goals>						<goal>single</goal>
					</goals>
					<configuration>						<descriptorRefs>
							<descriptorRef>jar-with-dependencies</descriptorRef>						</descriptorRefs>
						<archive>							<manifest>
								<mainClass>org.sonatype.mavenbook.App</mainClass>							</manifest>
						</archive>					</configuration>
				</execution>			</executions>
		</plugin>	</plugins>
</build>
```
现在就可以使用mvn package生成可运行jar文件。

## 套件描述符概述
