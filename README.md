# Gradle 配置文件标准化

## 1. 项目根目录中的build.gradle 中引入上面两个文件：

[下载build.gradle实例文件](https://raw.githubusercontent.com/sxfad/gradle-scripts/master/build.gradle)
 

	/**
	常用命令：
	1. tasks 列出项目可运行的task
	2. assemble - 打包输出项目.
	3. bootRepackage - 打可以通过'java -jar' 命令执行的JAR或WAR包
	4. build - 测试并打包输出.
	5. clean - 删除build目录.
	6. jar - 打jar包.
	7. javadoc - 打javadoc包.
	8. dependencies - 显示项目的所有依赖关系.
	9. uploadArchives - 上传':archives'中配置的所有包到maven 仓库
	*/
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
	subprojects {
	    // 参考 https://github.com/sxfad/gradle-scripts
	    apply from: sxGradleHome + 'maven.gradle'
	    apply from: sxGradleHome + 'spring-cloud.gradle'
	
	    version = '0.0.1' // 设置版本
	    group = 'com.suixingpay.demo' // 设置group id
	    description = 'demo' // 设置描述
	
	    dependencies {
	        compileOnly('org.springframework.boot:spring-boot-configuration-processor')
	        compileOnly('org.projectlombok:lombok')
	        testCompile('org.projectlombok:lombok')
	        testCompile("org.springframework.boot:spring-boot-starter-test")
	    }
	}


## 2. 子项目中只维护依赖及版本等信息即可，例如：

    bootRepackage {
        enabled = true  // 如果需要打spring boot 可执行jar，需要设置为true, 在spring-cloud.gradle 文件中已默认设置为false
    }
    dependencies {
        ... ...
    }
    version = '1.0.2' // 设置版本号
