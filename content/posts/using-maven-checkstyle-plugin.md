+++
title = "Using Maven Checkstyle Plugin"
date = "2013-06-21"
slug = "2013/06/21/using-maven-checkstyle-plugin"
Categories = []
+++

## 为什么要统一代码风格？

每个人都有自己喜欢的代码风格，萝卜生菜各有所爱，属于个人的喜好问题。就像穿衣打扮一样，有些人按照自己的审美观来打扮自己，
也有些人是为了凸显自己的与众不同与个性，还有些不爱收拾、怎么舒服怎么方便就怎么穿的。正是因为不同，我们的世界才是多姿多彩。

在你的团队里面，如果每个人写代码的时候都按照自己的个性去写，结果一定很有趣，就像一个人写的字一样，一看就知道是张三的字；
这段代码一看就是李四的杰作，根本不需要在每个文件头注明 `@author` ！

好吧，我并不觉得有趣，我的智商还不足以在面对不同风格的代码之间频繁切换的同时，还能集中精力去思考业务和逻辑问题，
我认为整洁的代码能够降低我的理解成本，从而用宝贵的时间去解决真正有价值的问题。

## 编码规范与执行官

有一些开发团队也认为统一的代码风格很重要，意识到必须采用一些方法来规范所有的写代码的习惯，因此整理出一份符合团队习惯的
《编码规范》，可能是以 Microsoft Word 或者 PDF 的形式放在公共的文档服务器中，也可能是在开发团队内部 Wiki 上面一个
Page，总之，这是一份指导大家如果编码的强制性的规范。

有了规范以后，怎样保证大家能够在日常的开发过程中严格遵守编码规范呢？这种事情靠大家自觉当然是不行的，都这么自觉的话，Team
Leader 还有什么用？那是要丢饭碗的！所以他们必须承担执行官的责任，监督大家提交的代码是否符合规范，一旦发现有不上道的，
一定要在后面抽两鞭子，丫的，有规范竟然不执行！

可是执行官还有别的事情要做啊，要看看有没有偷懒的、进度压得够不够紧、工作是否饱满，还有别的一堆日常规范要监督，可忙了；
而且，一个人去 Review 所有其他人的代码，这种事情还是挺累的，没办法只有降低频率，并且采取抽查的方式，这样依赖，
每个开发人员的代码在一个月能被 Review 到一次就不错了。

## 自动化的代码风格检查

其实我们的 Team Leader 还是很靠谱的，他意识到这样下去不是办法，而且我们是优秀的程序员，难道就不能写个程序来自动地对
大家的代码进行检查吗？大家每次提交代码之前，先运行一次检查程序，没问题了再提交。图灵说了，所有能通过既定的步骤解决的问题，
都可以用图灵机来解决，目前我们所用到的计算机都是图灵机。

幸好，[Checkstyle](http://checkstyle.sourceforge.net/) 的作者们已经完成了这项工作，通过一个 Maven 的
[Plugin](http://maven.apache.org/plugins/maven-checkstyle-plugin/) 可以很方便的集成到我们日常的构建过程中：

### Maven Checkstyle Plugin 的使用

首先，在你的 `pom.xml` 里加入 `maven-checkstyle-plugin` 的配置

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-checkstyle-plugin</artifactId>
	<version>2.10</version>
</plugin>
```

修改配置，使用自定义的 Checkstyle 配置文件；其中，`${project.basedir}`是指项目的根目录：

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-checkstyle-plugin</artifactId>
	<version>2.10</version>
	<configuration>
		<configLocation>${project.basedir}/conf/checkstyle.xml</configLocation>
	</configuration>
</plugin>
```

最后，我们还需要在检查到代码风格有问题的时候，让我们的构建过程失败：

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-checkstyle-plugin</artifactId>
	<version>2.10</version>
	<executions>
		<execution>
			<phase>process-sources</phase>
			<goals>
				<goal>check</goal>
			</goals>
		</execution>
	</executions>
	<configuration>
		<failsOnError>true</failsOnError>
		<configLocation>${project.basedir}/conf/checkstyle.xml</configLocation>
	</configuration>
</plugin>
```

检查的错误信息可以直接查看生成的 `checkstyle-results.xml` 文件：

```bash
$ less target/checkstyle-results.xml
```

