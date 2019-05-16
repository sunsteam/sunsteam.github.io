---
layout:     post
title:      "Gradle 打包依赖为 fatJar 添加源码上传到 Maven"
subtitle:   ""
date:       2019-05-16 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Java
    - Gradle
    - Maven
---


本文记录内容：Gradle 编译，打 jar 包的时候如果遇到有依赖库只有本地 jar 包，不提供在线仓库依赖的时候，如何把所有依赖打包在一起，附带自己的源码一起上传到 maven 仓库

Gradle: 4.10
Java:   1.8

## 1. 合并本地依赖 jar 包，打包出 fatJar

### 1.1 首先贴一下项目结构

```groovy

buildscript {

    ext {
        nexusConfig = [
                "repository"      : "https://repo.xxxx",
                "uploaderName"    : "xxx",
                "uploaderPassword": "xxxxxx",
                "readerName"      : "xxx",
                "readerPassword"  : "xxxxxx"
        ]
    }


    dependencies {classpath "com.github.jengelman.gradle.plugins:shadow:4.0.4"}

    repositories {
        jcenter()
    }
}

subprojects {
    apply plugin: 'java'
    sourceCompatibility = 1.8
    group 'com.xxxx.xxxxxx'

    repositories {
        jcenter()
        mavenCentral()
        flatDir { dirs 'libs' }
        maven {
            url nexusConfig.repository
            credentials {
                username nexusConfig.readerName
                password nexusConfig.readerPassword
            }
        }
    }

    dependencies {
        testImplementation 'junit:junit:4.12'
    }
}
```

**Project 配置文件**

```groovy

version '1.1.2'
description 'xiaomi push service'

dependencies {
    implementation fileTree('libs')
}


apply from: "../publish.gradle"
apply from: "../shadow.gradle"

```

如上, 这是一个使用小米推送的服务端业务封装库，小米只提供了本地 jar 包，需要打包在一起供其他项目依赖。


### 1.2 使用 Shadow 插件打 fatJar

[插件地址](https://plugins.gradle.org/plugin/com.github.johnrengelman.shadow#kotlin-usage)

shadow.gradle 的内容

```groovy

apply plugin: 'com.github.johnrengelman.shadow'

shadowJar {
    // 完整名称为 baseName-version-classifier.jar
    baseName = project.name
    // 默认为 '-all' 为 null 则去除该参数
    classifier = null
    version = project.version

    // 方法数超过 65535 会报错, 需要打开下面这个配置
    //zip64 = true

    // 去除和添加文件 META-INF
    // include '.... 文件'
    // exclude '... 文件'
    
    // 如果有 Main 函数, 如下配置启动类
    //manifest {
    //    attributes 'Main-Class': 'com.example.Main'
    //}
}

sourcesJar.dependsOn(shadowJar)

```

### 1.3 Shadow 打包原理

打 fatJar 有好几种方式，Shadow 用的是第二种

1. 解压所有 jar，重新压缩合并在一个 jar 中
2. 不合并，只是把依赖 jar 移动到最终 jar 包的 lib 目录下，然后在 manifest 中把 class-path 指向这个地址，那么加载类时就能正确找到
3. 嵌套 jars, SpringBoot Gradle plugin 在用，需要用它的启动器自定义 ClassLoader 来启动，会加入很多业务无关代码

**优点**：
- 结构和原理简单
- 不会有无关代码

**缺点**：
- 类文件路径被改了， 因此如果有直接调用 Class 文件名字或路径进行类加载的代码，会报错

### 1.4 Shadow 实现

插件会创建 2 个 task, shadowJar 负责打包， uploadShadow 负责上传 Maven，核心代码就是下面这一段了


```groovy

    protected void configureShadowTask(Project project) {
        JavaPluginConvention convention = project.convention.getPlugin(JavaPluginConvention)
        ShadowJar shadow = project.tasks.create(SHADOW_JAR_TASK_NAME, ShadowJar)
        shadow.group = SHADOW_GROUP
        shadow.description = 'Create a combined JAR of project and runtime dependencies'
        shadow.conventionMapping.with {
            map('classifier') {
                'all'
            }
        }
        if (GradleVersion.current() >= GradleVersion.version("5.1")) {
            shadow.archiveClassifier.set("all")
        }
        shadow.manifest.inheritFrom project.tasks.jar.manifest
        shadow.doFirst {
            def files = project.configurations.findByName(ShadowBasePlugin.CONFIGURATION_NAME).files
            if (files) {
                def libs = [project.tasks.jar.manifest.attributes.get('Class-Path')]
                libs.addAll files.collect { "${it.name}" }
                manifest.attributes 'Class-Path': libs.findAll { it }.join(' ')
            }
        }
        shadow.from(convention.sourceSets.main.output)
        shadow.configurations = [project.configurations.findByName('runtimeClasspath') ?
                                         project.configurations.runtimeClasspath : project.configurations.runtime]
        shadow.exclude('META-INF/INDEX.LIST', 'META-INF/*.SF', 'META-INF/*.DSA', 'META-INF/*.RSA', 'module-info.class')

        project.artifacts.add(ShadowBasePlugin.CONFIGURATION_NAME, shadow)
        configureShadowUpload()
    }

    private void configureShadowUpload() {
        configurationActionContainer.add(new Action<Project>() {
            void execute(Project project) {
                project.plugins.withType(MavenPlugin) {
                    Upload upload = project.tasks.withType(Upload).findByName(SHADOW_UPLOAD_TASK)
                    if (!upload) {
                        return
                    }
                    upload.configuration = project.configurations.shadow
                    MavenPom pom = upload.repositories.mavenDeployer.pom
                    pom.scopeMappings.mappings.remove(project.configurations.compile)
                    pom.scopeMappings.mappings.remove(project.configurations.runtime)
                    pom.scopeMappings.addMapping(MavenPlugin.RUNTIME_PRIORITY, project.configurations.shadow, Conf2ScopeMappingContainer.RUNTIME)
                }
            }
        })
    }

```

上面的代码就是配置 task 实现了 jar 包位置替换， 指定 `Class-Path`，移除多余文件，去除 Maven 中的打包任务替换为自己的然后上传

[更多代码看 GitHub](https://github.com/johnrengelman/shadow)

## 2. 上传源码到 Maven

按照第一节的内容，已经完成了打包 fatJar 并上传 Maven 的操作，但我发现使用 uploadShadow 这个 Shadow 插件自带的 task 上传的 jar 包里，不带源码！不爽，看看能不能改进。

### 2.1 不使用 uploadShadow 任务，但调用 shadowJar 任务做打包

实现方案如上述标题，默认的 Maven 插件提供的 task 中会调用 jar 这个 task 进行打包，如果有指定源码，那么也会同步上传源码，uploadShadow 中把实现给替换了，但没有写入上传源码的操作，就算打出源码也不会上传，所以换种思路。


```groovy

apply plugin: 'maven'

task sourcesJar(type: Jar) {
    classifier = 'sources'
    from sourceSets.main.java.srcDirs
}

artifacts {
    archives sourcesJar
}

// 如果希望 gradle install，安装到. m2 本地仓库，参考下面的内容
install {
    repositories.mavenInstaller {
        pom.project {
            version project.version
            artifactId project.name
            groupId project.group
            packaging 'jar'
            description project.description
        }
    }
}

uploadArchives {
    configuration = configurations.archives
    repositories {
        mavenDeployer {
            repository(url: nexusConfig.repository) {
                authentication(
                        userName: nexusConfig.uploaderName,
                        password: nexusConfig.uploaderPassword
                )
            }

            snapshotRepository(url: nexusConfig.repositorySnapshot) {
                authentication(
                        userName: nexusConfig.snapshotName,
                        password: nexusConfig.snapshotPassword
                )
            }

            pom.project {
                version project.version
                artifactId project.name
                groupId project.group
                packaging 'jar'
                description project.description
            }
        }
    }
}

```


上面这个 publish.gradle 是无 shadow 依赖的, 只是加入一个全局的 sourceJar 任务用来生成源码。

configurations.archives 是 uploadArchives 任务的默认实现，会上传源码，因此我们只要让 sourceJar 和 shadowJar 这 2 个任务都运行一遍就可以了。

只要 shadow.gradle 在 publish.gradle 之后加载，它就能拿到 publish.gradle 中定义的 sourceJar 任务，并指定这个任务依赖 shadowJar 运行，uploadArchives 在进行编译源码操作时会连带先调用 shadowJar 编译 fatJar 替换掉 jar 任务的编译结果，最后一起上传，简单并解耦的实现了打包 fatJar 并带源码上传 Maven 的需求

```groovy


apply from: "../publish.gradle"
apply from: "../shadow.gradle"

```

shadow.gradle

```groovy
apply plugin: 'com.github.johnrengelman.shadow'

shadowJar {
    .....
}

sourcesJar.dependsOn(shadowJar)

```