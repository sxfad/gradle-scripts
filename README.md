# 有了Gradle，还会选Maven吗？


Gradle 在随行付标准化实践：一行代码带来的变革！

---

现在许多人还在为使用Maven 还是 Gradle 而纠结。如果关注过《Maven权威指南》作者许晓斌老师在InfoQ中发表的文章：[《Maven实战（六）——Gradle，构建工具的未来？》](http://www.infoq.com/cn/news/2011/04/xxb-maven-6-gradle)，那么一定会有同感：Gradle太灵活，可能会造成不可控。文章中的原话是：

“Gradle的另外一个问题就是它太灵活了，虽然它支持约定优于配置，不过从本文你也看到了，破坏约定是多么容易的事情。人都喜欢自由，爱自定义，觉得自己的需求是多么的特别，可事实上，从Maven的流行来看，几乎95%以上的情况你不需要自行扩展，如果你这么做了，只会让构建变得难以理解。从这个角度来看，自由是把双刃剑，Gradle给了你足够的自由，约定优于配置只是它的一个选项而已，这初看起来很诱人，却也可能使其重蹈Ant的覆辙。”


首先看一下我们最初使用Gradle构建Spring Cloud项目的build.gradle的写法：

	buildscript {
	    ext {
	        springCloudVersion = 'Edgware.SR3'
	        springBootVersion = '1.5.9.RELEASE'
	        REPOSITORY_HOME = "http://maven.aliyun.com"
	    }
	    repositories {
	        maven { url "${REPOSITORY_HOME}/nexus/content/groups/public" }
	    }
	    dependencies {
	        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
	    }
	}
	apply plugin: 'maven'
	apply plugin: 'java'
	apply plugin: 'org.springframework.boot'
	
	if (JavaVersion.current().isJava8Compatible()) {
	    allprojects {
	        tasks.withType(Javadoc) {
	            options.encoding = 'UTF-8'
	            options.addStringOption('Xdoclint:none', '-quiet') // 关闭JDK1.8的doclint特性
	        }
	    }
	}
	// 导入Spring Cloud 依赖
	dependencyManagement {
	  imports {
	    mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
	  }
	}
	
	dependencyManagement {
	  resolutionStrategy {
	    // 检查远程依赖是否存在更新
	    cacheChangingModulesFor 0, 'seconds'
	    cacheChangingModulesFor 0, 'seconds' // 修改本地缓存策略
	  }
	}
	
	sourceCompatibility = JavaVersion.VERSION_1_8
	targetCompatibility = JavaVersion.VERSION_1_8
	
	bootRepackage {
	  // 默认只打普通jar包
	  enabled = false
	}
	
	// 打包源代码，为了方便查看源码及调试，把源码也上传到nexus仓库中
	task sourcesJar(type: Jar) {
	  classifier = 'sources'
	  from sourceSets.main.allSource
	}
	
	// 打javadoc包，为了方便查看注释，需要把javadoc也上传到nexus仓库中
	task javadocJar(type: Jar, dependsOn: javadoc) {
	  classifier = 'javadoc'
	  from javadoc.destinationDir
	}
	
	artifacts {
	  archives jar
	  archives sourcesJar
	  archives javadocJar
	}
	
	uploadArchives {
	    repositories {
	        mavenDeployer {
	            snapshotRepository(url: "${REPOSITORY_HOME}/nexus/content/repositories/snapshots/") {
	                authentication(userName: 'xxx', password: 'xxx')
	            }
	            repository(url: "${REPOSITORY_HOME}/nexus/content/repositories/releases/") {
	                authentication(userName: 'xxx', password: 'xxx')
	            }
	        }
	    }
	}
	
	version = '0.0.1' // 设置版本
	group = 'com.suixingpay.demo' // 设置group id
	description = 'demo' // 设置描述
	
	dependencies {
	    compile('org.springframework.boot:spring-boot-starter-actuator')
	    compile('org.springframework.boot:spring-boot-starter-web')
	    compileOnly('org.springframework.boot:spring-boot-configuration-processor')
	    compileOnly('org.projectlombok:lombok')
	    testCompile('org.projectlombok:lombok')
	    testCompile "org.springframework.boot:spring-boot-starter-test"
	}

通过上面代码我们可以看出以下问题：

1. 虽然dependencies部分代码比maven少了许多，但总体来看，代码并不简洁；
2. 存在硬编码带来的风险，比如要修改Maven仓库的地址，或用户名及密码时，需要通知所有人进行变更；
3. gradle代码除buildscript代码块外，没有像maven中xml一样结构性书写要求，那么可能造成每个人的写法是不一样的情况；
4. 很难区分每个插件的配置都有哪些，对于不熟悉插件的人来说是非常痛苦的事情，非常不方便维护；
5. 当项目多时，会有非常多的重复代码，增加了非常多的重复开发及维护成本；比如之前没有要求将源码及javadoc上传到maven仓库中，后来因各种原因，增加了这项需求，那么就需要通知所有把所有代码都修改一遍；

我们最初的想法是将上面代码，根据不同的插件，使用不同的gradle文件来维护，再将它们放到GRADLE_HOME/init.d/目录下，但还是更新难的问题。

补充说明init.gradle的加载顺序，Gradle会依次对一些目录进行检测，按照优先级加载这些目录下的文件，如果一个目录下有多个文件被找到，则按照英文字母的顺序依次加载。加载优先级如下：

1. 通过 -I 或者 –init-script 参数在构建开始时指定路径，如
  
        gradle --init-script init.gradle clean
    
        gradle --I init.gradle assembleDebug
    
2. 加载USER_HOME/.gradle/init.gradle文件

3. 加载USER_HOME/.gradle/init.d/目录下的以.gradle结尾的文件

4. 加载GRADLE_HOME/init.d/目录下的以.gradle结尾的文件

后来研究发现gradle 可以通过 apply from 命令加载外部gradle文件，甚至可以加载远程http服务器中的文件。这个发现让我们兴奋不已，有了它可以帮助我们解决上面所有的问题。

首先将上面build.gradle的内容拆分成多个文件：

1. 将maven相关的代码放入[maven.gradle](https://github.com/sxfad/gradle-scripts/blob/master/maven.gradle)文件中；
2. 因java、spring boot和Spring Cloud是我们平时用得最多的插件，所以我们将它们都相关的代码放到了同一个文件中：[spring-cloud.gradle](https://github.com/sxfad/gradle-scripts/blob/master/spring-cloud.gradle)，但Spring boot及Spring Cloud的版本会因为项目的不同而使用不同的版本，所以需要将它们的单独提取出来，比如：[spring-cloud-dalston-sr4.gradle](https://github.com/sxfad/gradle-scripts/blob/master/spring-cloud-dalston-sr4.gradle)、[spring-cloud-edgware.gradle](https://github.com/sxfad/gradle-scripts/blob/master/spring-cloud-edgware.gradle)

然后将上面拆分好的文件入到git仓库中，并修改build.gradle文件：

	buildscript { // buildscript 不能抽取出来，只能重复写。
	  ext{
	    sxGradleHome = "https://raw.githubusercontent.com/sxfad/gradle-scripts/master/"
	  }
	  apply from: sxGradleHome + 'maven.gradle'
	  apply from: sxGradleHome + 'spring-cloud-edgware.gradle' // 导入使用Spring Cloud及相应的Spring Boot版本号
	  repositories {
	    maven { url REPOSITORY_URL }
	  }
	  dependencies {
	    classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
	  }
	}

    // 参考 https://github.com/sxfad/gradle-scripts
    apply from: sxGradleHome + 'maven.gradle'
    apply from: sxGradleHome + 'spring-cloud.gradle'
	
    version = '0.0.1' // 设置版本
    group = 'com.suixingpay.demo' // 设置group id
    description = 'demo' // 设置描述
	
	dependencies {
	    compile('org.springframework.boot:spring-boot-starter-actuator')
	    compile('org.springframework.boot:spring-boot-starter-web')
	    compileOnly('org.springframework.boot:spring-boot-configuration-processor')
	    compileOnly('org.projectlombok:lombok')
	    testCompile('org.projectlombok:lombok')
	    testCompile "org.springframework.boot:spring-boot-starter-test"
	}
	
通过上面的方法处理后，给我们带来如下好处：

1. 修改后的build.gradle变得极其简洁，只需要关心其基本信息及依赖即可；
2. 对于公司内部使用来说，让gradle的使用更加规范和标准，也使用变得更加简单；
3. 因为核心gradle文件放到了远程服务器中，非常方便更新和维护；并且使用git管理后，还带有“版本管理”功能（方便查看文件修改历史，及回退等）；


看到这之后，你一定不会再为选择Maven还是Gradle纠结，甚至已经爱上了Gradle。
