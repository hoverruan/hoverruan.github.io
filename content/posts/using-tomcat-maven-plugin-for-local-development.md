+++
title = "Using Tomcat Maven Plugin for Local Development"
date = "2013-01-06"
slug = "2013/01/06/using-tomcat-maven-plugin-for-local-development"
Categories = []
+++

## 手动配置Tomcat服务器

在日常开发的过程中，我们经常需要配置一个本地的服务器进行本地的开发测试环境；以Tomcat服务器为例，配置的过程最少包含这样的一些步骤：

1. 下载并安装Tomcat服务器
2. 在IDE中配置一个本地Tomcat的运行环境
3. 用配置好的Tomcat运行环境来对项目进行开发和测试

如果你的项目使用Maven来进行管理，则每次你的pom.xml文件发生修改之后，IntelliJ IDEA会重新导入项目，原来对Web项目的特殊配置就会被重置成默认值，必须重新进行配置。

## 使用Tomcat Maven Plugin

Apache Tomcat Maven Plugin允许直接运行一个Web项目，无需额外下载安装Tomcat服务器，也无需在IDE中进行额外的配置，每次只需要一个简单的Maven命令即可启动一个Tomcat服务进行开发或者调试：

``` xml
<plugins>
	<plugin>
		<groupId>org.apache.tomcat.maven</groupId>
		<artifactId>tomcat6-maven-plugin</artifactId>
		<version>2.0</version>
		<configuration>
			<path>/${project.build.finalName}</path>
			<additionalClasspathDirs>
				<additionalClasspathDir>${project.basedir}/build/local</additionalClasspathDir>
			</additionalClasspathDirs>
		</configuration>
	</plugin>
</plugins>
```
只需执行以下命令即可启动服务：

```
mvn tomcat6:run
```

当然，可以在IDE里面配置执行此Maven Goal，可以方便地启动或者调试。
