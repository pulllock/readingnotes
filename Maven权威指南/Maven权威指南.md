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