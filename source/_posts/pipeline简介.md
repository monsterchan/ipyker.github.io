layout: post
title: pipeline语法初认识
author: Pyker
categories: CI/CD
tags:
  - jenkins
  - pipeline
date: 2019-08-30 20:43:00
---

# 1. pipeline简介

通俗的讲`Jenkins pipeline`是一组插件，支持将连续交付的实现和持续集成到Jenkins中。个人理解就是以前你所有创建的类似自由风格或者maven风格的项目都可以用pipeline流水线进行集成。



##  1.1 语法介绍

pipeline支持`声明式`和`脚本化`的pipeline语法，声明式和脚本化的流水线从根本上是不同的。 声明式流水线的是 Jenkins 流水线更近的特性:

* 相比脚本化的流水线语法，它提供更丰富的语法特性,
* 是为了使编写和读取流水线代码更容易而设计的。

然而通常我们在写Jenkinsfile的时候，都是声明式和脚本化相结合的流水线。但建议使用`声明式 Pipeline`的方式进行编写,从jenkins社区的动向来看，很明显这种语法结构也会是未来的趋势。



## 1.2 声明式pipeline

`声明式pipeline`是Jenkins Pipeline 的一个相对较新的补充， 它在Pipeline子系统之上提出了一种更为简化和有意义的语法。

所有有效的`声明式pipeline`必须包含在一个pipeline块内，例如：

```json
pipeline {
    agent any // 在任何可用的代理上，执行流水线或它的所有stage
    stages {
        stage('Build') { // 定义 "Build" 阶段
            steps {
                // 执行与 "Build" 阶段相关的步骤
            }
        }
        stage('Test') { // 定义"Test" 阶段
            steps {
                // 执行与"Test" 阶段相关的步骤
            }
        }
        stage('Deploy') { // 定义 "Deploy" 阶段
            steps {
                // 执行与 "Deploy" 阶段相关的步骤
            }
        }
    }
}
```

## 1.3 脚本化pipeline

在脚本化流水线语法中, 一个或多个 `node` 块在整个流水线中执行核心工作。

```json
node {  // 在任何可用的代理上，执行流水线或它的所有stage
    stage('Build') {  // 定义 "Build" 阶段
        //  执行与 "Build" 阶段相关的步骤
    }
    stage('Test') { // 定义 "Test" 阶段
        //  执行与 "Test" 阶段相关的步骤
    }
    stage('Deploy') { // 定义 "Deploy" 阶段
        // 执行与 "Deploy" 阶段相关的步骤
    }
}
```

> 本文语法将主要围绕声明式pipeline进行说明。

## 1.4 pipeline概念

在我们介绍pipeline指令之前，我们先来介绍一下pipeline的几个概念，以方便我们后续对pipeline指令的理解。

* `node` :  节点是一个机器 ，它是Jenkins环境的一部分，能够执行流水线。
* `stage`：stage 块定义了通过整个流水线执行的任务在层次上不同的子集(例如，“build”、“test”和“deploy”阶段)，许多插件使用这些阶段来可视化或显示Jenkins管道状态/进度。
* `steps` :  本质上 ，它是一个单一的任务,  steps 告诉Jenkins 在特定的时间点要做什么事 。 举个例子,要执行shell命令 ，请使用 `sh` 步骤: `sh 'make'`。

# 2. pipeline指令

## 2.1 agent

该agent部分指定整个Pipeline或特定阶段将在Jenkins环境中执行的位置，具体取决于该agent 部分的放置位置。该部分必须在pipeline块内的顶层定义 ，但stage级使用是可选的。

为了支持Pipeline可能拥有的各种用例，该agent部分支持以下几种不同类型的参数。这些参数可以应用于pipeline块的顶层，也可以应用在每个stage指令内。

* `any`： 在任何可用的agent 上执行Pipeline或stage。如：`agent any`

* `none`： 当pipeline块顶层使用none时，那么整个Pipeline不会分配全局agent ，而是每个stage将会包含其自己的agent部分。

* `label`：在Jenkins环境中使用提供的label标签代理上执行Pipeline或stage。如：`agent { label 'my-defined-label' }`

* `node`： 它和label标签一样，`agent { node { label 'labelName' } }` 等同于`agent { label 'labelName' }`

* `docker`： 使用给定的容器执行整个流水线或某阶段，该容器将在预先配置为接受该容器的节点上或者匹配到的label上进行动态供应。docker还可以接受一个`args参数`，该参数可能包含直接传递给`docker run`调用的参数，以及一个`alwaysPull`选项，该选项将强制`docker pull`，即使镜像名称已经存在。或docker还可以选择接受`registryUrl`和`registryCredentialsId`参数，这些参数将有助于指定要使用的Docker私仓及凭证。

  ```yaml
  agent {
      docker {
          image 'maven:3-alpine'
          label 'my-defined-label'
          args  '-v /tmp:/tmp'
          registryUrl 'https://myregistry.com/'
          registryCredentialsId 'myPredefinedCredentialsInJenkins'
      }
  }
  ```

* `dockerfile`： 使用从Dockerfile源存储库中包含的容器来构建执行Pipeline或stage，为了使用此选项，Jenkinsfile必须使用`Multibranch Pipeline`或`Pipeline from SCM`加载。默认是在Dockerfile源库的根目录：agent { dockerfile true }。如果在另一个目录中构建Dockerfile，使用dir 选项：`agent { Dockerfile { dir 'someSubDir' }}`。如果您的Dockerfile有另一个名字，可以使用filename选项指定文件名。还可以使用additionalBuildArgs选项向docker build…命令传递额外的参数，比如代理`{dockerfile {additionalBuildArgs’——build-arg foo=bar’}}`。同样dockerfile还可以接受`registryUrl`和`registryCredentialsId`参数，这些参数将有助于指定要使用的Docker私仓及凭证。

  ```yaml
  agent {
      // 相当于 "docker build -f Dockerfile.build --build-arg version=1.0.2 ./build/
      dockerfile {
          filename 'Dockerfile.build'
          dir 'build'
          label 'my-defined-label'
          additionalBuildArgs  '--build-arg version=1.0.2'
          args '-v /tmp:/tmp'
      }
  }
  ```

* `kubernetes`： 执行流水线或阶段部署在Kubernetes集群上的Pod内。为了使用此选项，必须从多分支管道或“SCM管道”加载Jenkinsfile。Pod模板是在`kubernetes{ }`块中定义的。例如，如果您想要一个包含Kaniko容器的pod。

  ```yaml
  agent {
      kubernetes {
          label podlabel
          yaml """
  kind: Pod
  metadata:
    name: jenkins-slave
  spec:
    containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      imagePullPolicy: Always
      command:
      - /busybox/cat
      tty: true
      volumeMounts:
        - name: aws-secret
          mountPath: /root/.aws/
        - name: docker-registry-config
          mountPath: /kaniko/.docker
    restartPolicy: Never
    volumes:
      - name: aws-secret
        secret:
          secretName: aws-secret
      - name: docker-registry-config
        configMap:
          name: docker-registry-config
  """
     }
  ```

  > * 声明式管道需要Jenkins 2.66+才支持
  >
  > * 更多关于jenkins kubernetes插件使用，参考：https://github.com/jenkinsci/kubernetes-plugin

## 2.2 常见选项

以下选项可以应用于两个或多个agent实现。除非明确说明，否则一般不需要它们。

* `label`： 一个字符串。用于标记在哪里运行管道或单个阶段的标签。
* `customWorkspace`： 一个字符串。在agent上自定义工作区来运行pipeline或单个stage
* `reuseNode`： 默认为false。如果为真，则在管道顶层指定的节点的相同的工作区上运行容器，而不是在新节点上运行容器。仅在 单个stage中使用agent才有效。

* `args`： 一个字符串。要传递给docker run的运行时参数。

## 2.3 post

定义Pipeline或stage运行结束时的操作。post-condition块支持post部件：`always`, `changed`, `failure`, `success`, `unstable`, 和 `aborted`。这些块允许在Pipeline或stage运行结束时执行步骤，具体取决于Pipeline的状态。

**conditions**

* `always`：无论流水线或阶段的完成状态如何，都允许在 `post` 部分运行该步骤。
* `changed`：只有当前流水线或阶段的完成状态与它之前的运行不同时，才允许在 `post` 部分运行该步骤。
* `failure`：只有当前流水线或阶段的完成状态为"failure"，才允许在 `post` 部分运行该步骤。
* `success`：只有当前流水线或阶段的完成状态为"success"，才允许在 `post` 部分运行该步骤。
* `unstable`：只有当前流水线或阶段的完成状态为"unstable"，才允许在 `post` 部分运行该步骤。
* `aborted`：只有当前流水线或阶段的完成状态为"aborted"，才允许在 `post` 部分运行该步骤。
* `cleanup`： 无论流水线或阶段的状态如何，在评估完所有其他post条件之后，在此post条件中运行步骤。
* `unsuccessful`：只有当前流水线或阶段的运行不为“成功”状态时，才在post中运行这些步骤

* `regression`： 只有当前流水线或阶段运行状态为`failure`、`unstable`或`aborted`，且上一次运行`success`时，才在post中运行这些步骤。

* `fixed`： 只有当前流水线或阶段的运行成功，而前一个运行失败或不稳定时，才在post中运行这些步骤。

  

> 在这些conditions中，我们比较常用的就是`success`和`failure`了。通常用于构建成功或者失败发送构建通知的。

```yaml
// 以下是一个用企业微信触发构建完成后成功和失败的通知
post {
        success {
            script {
                sh("curl -s -X POST 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxxxx' \
                -H 'Content-Type: application/json' \
                -d '{\"msgtype\": \"markdown\",\"markdown\": {\"content\": \"### <font color=#00FF00>Build Success！！ </font> \n ”}}'")
            }
        }
        failure {
            script {
                sh("curl -s -X POST 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxxxx' \
                -H 'Content-Type: application/json' \
                -d '{\"msgtype\": \"markdown\",\"markdown\": {\"content\": \"### <font color=#FF0000>Build failed！！ </font>  \n "}}'")
            }
        }
    }
```

## 2.4 stages

包含`一个或多个stage`指令的序列，Pipeline的大部分工作在此执行。建议stages至少包含至少一个stage指令，用于连接各个交付过程，如构建，测试和部署等。

```yaml
pipeline {
    agent any
    stages { 
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

## 2.5 steps

`steps`部分定义了一系列要在给定阶段指令中执行的一个或多个步骤。

```yaml
pipeline {
    agent any
    stages { 
        stage('Example') {
            steps {
            	sh "curl http://www.baidu.com"
                echo 'Hello World'
            }
        }
    }
}
```

## 2.6 environment

`environment`指令指定一系列键值对，这些键值对将被定义为`所有steps`或`指定stage中steps`的环境变量，具体取决于environment指令在Pipeline中的位置。该指令支持一种特殊的方法`credentials()`，可以通过其在Jenkins环境中的标识符来访问预定义的凭据。

**支持的凭据类型**

* `Secret Text`：指定的环境变量将被设置为秘密文本内容。输出结果为密码格式的点点点。
* `Secret File`：指定的环境变量将被设置到临时创建的文件的位置
* `Username and password`：指定的环境变量将设置为username:password，另外两个环境变量将自动定义:`MYVARNAME_USR`和`MYVARNAME_PSW`。
* `SSH with Private Key`：指定的环境变量将被设置到临时创建的SSH密钥文件的位置，还可以自动定义两个额外的环境变量:`MYVARNAME_USR`和`MYVARNAME_PSW`(保存密码)。

```yaml
pipeline {
    agent any
    environment { # 顶级流水线块中使用的环境指令将应用于pipeline中的所有stage
        CC = 'clang'
    }
    stages {
        stage('Example') {
            environment { # 在阶段中定义的环境指令只将给定的环境变量应用于阶段中的steps
                AN_ACCESS_KEY = credentials('my-prefined-secret-text') 
                # environment块有一个帮助方法credentials()，可以使用其在Jenkins环境中的标识符访问预定义凭证。
            }
            steps {
                sh 'printenv'
            }
        }
    }
}
```

## 2.7 options

`options` 指令允许从流水线内部配置特定于流水线的选项。 流水线提供了许多这样的选项, 比如 `buildDiscarder`,但也可以由插件提供, 比如 `timestamps`, 如：

```json
options {
  buildDiscarder(logRotator(numToKeepStr: '1'))  // pipeline保持构建的最大个数
  checkoutToSubdirectory('foo') // 在工作区的子目录中执行自动源代码控制检出
  disableConcurrentBuilds() // 不允许并行执行Pipeline,可用于防止同时访问共享资源等
  skipDefaultCheckout() // 默认跳过来自源代码控制的代码
  skipStagesAfterUnstable() // 一旦构建状态进入了"Unstable"状态，就跳过此stage
  timeout(time: 1, unit: 'HOURS') // 设置Pipeline运行的超时时间
  retry(3) // 失败后，重试整个Pipeline的次数
  timestamps() // 预定义由Pipeline生成的所有控制台输出时间
  quietPeriod(30) // 设置pipeline的静默期（以秒为单位），覆盖全局默认值
  disableResume() // 如果主程序重新启动，不允许恢复管道
  ...
}
```

### 2.7.1 stage options

`stage options`指令类似于pipeline顶层目录中的options指令。然而，`stage options`只能包含`retry`、`timeout`或`timestamps`等步骤，或者与阶段相关的声明性选项，如`skipDefaultCheckout`

## 2.8 parameters

parameters指令提供用户在触发管道时应该提供的参数列表。也就是平常使用的参数化构建。这些用户指定参数的值可通过params对象用于pipeline steps。

**可用的parameters**

* `string`：字符串类型的参数。如：`parameters { string(name: 'DEPLOY_ENV', defaultValue: 'staging', description: '') }`
* `text`：一个文本参数，它可以包含多行。如：`parameters { text(name: 'DEPLOY_TEXT', defaultValue: 'One\nTwo\nThree\n', description: '') }`
* `booleanParam`：一个布尔参数，如：`parameters { booleanParam(name: 'DEBUG_BUILD', defaultValue: true, description: '') }`
* `choice`：一个选择参数，如：`parameters { choice(name: 'CHOICES', choices: ['one', 'two', 'three'], description: '') }`
* `password`：一个密码参数，如：`parameters { password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'A secret password') }`

```yaml
pipeline {
    agent any
    parameters {
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')

        text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person')

        booleanParam(name: 'TOGGLE', defaultValue: true, description: 'Toggle this value')

        choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something')

        password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password')
    }
    stages {
        stage('Example') {
            steps {
                echo "Hello ${params.PERSON}"

                echo "Biography: ${params.BIOGRAPHY}"

                echo "Toggle: ${params.TOGGLE}"

                echo "Choice: ${params.CHOICE}"

                echo "Password: ${params.PASSWORD}"
            }
        }
    }
}
```

以上示例结果如下：

![](/images/pic/pipeline1.png)

## 2.9 triggers

`trigger指令`定义了应该重新触发管道的自动化方式。对于与源(如GitHub或BitBucket)集成的管道，可能不需要触发器，因为可能已经存在基于webhook的集成。目前可用的触发器有`cron`、`pollSCM`和`upstream`。

* `cron`： 接受cron样式的字符串，以定义应该重新触发管道的常规间隔，如: `trigger {cron('H */4 * * * 1-5')}`

* `pollSCM`：接受一个cron风格的字符串来定义Jenkins检查SCM源更改的常规间隔。如果存在新的更改，则Pipeline将被重新触发。如: `triggers { pollSCM('H */4 * * 1-5') }`
* `upstream`：接受逗号分隔的作业字符串和阈值。当字符串中的任何作业以最小阈值结束时，将重新触发pipeline。如：  `triggers { upstream(upstreamProjects: 'job1,job2', threshold: hudson.model.Result.SUCCESS) }`

> 由于正真使用jenkins的触发器时，我们一般使用gitlab的webhooks。很少使用cron或者pollscm，所以这里就先跳过。

## 2.10 tools

通过tools可自动安装工具，并放置环境变量到PATH。如果设置agent none，则忽略此操作。支持自动安装的工具有`maven`，`jdk`，`gradle`

```yaml
pipeline {
    agent any
    tools {
        maven 'apache-maven-3.0.1' 
    }
    stages {
        stage('Example') {
            steps {
                sh 'mvn --version'
            }
        }
    }
}
```



> 通常这个我们也不会自动安装，都是手动指定，`在Jenkins 管理Jenkins → 全局工具配置中预配置`

## 2.11 input

input指令允许你在stage中使用input step输入提示。在应用了任何options之后，以及在进入stage 、agent或评估其when条件之前，阶段将暂停。如果输入被批准，则阶段将继续。作为输入提交的一部分提供的任何参数都将在该阶段的其余部分的环境中可用。

**配置的选项**

* `message`：交互式构建显示给用户的提示信息。<必需项参数>
* `id`：此输入的可选标识符。默认为stage名称
* `ok`：交互式上“ok”按钮的文本样式
* `submitter`：允许提交此输入的用户或外部组名的可选逗号分隔列表。默认允许任何用户
* `submitterParameter`：环境变量的可选名称，如果存在，则使用提交者名称进行设置。
* `parameters`：一个可选的参数列表。（上文已经说过）

```yaml
pipeline {
  agent any
  stages {
    stage('Example') {
      input {
        message "Should we continue?"
        ok "Yes, we should."
        submitter "alice,bob"
        parameters {
          string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
             }
          }
      steps {
         cho "Hello, ${PERSON}, nice to meet you."
            }
        }
    }
}
```

构建时效果如下：

![](/images/pic/pipeline2.png)

## 2.12 when

`when指令`允许Pipeline根据给定的条件确定是否执行该阶段。该when指令必须至少包含一个条件。如果when指令包含多个条件，则所有子条件必须为stage执行返回true。这与子条件嵌套在一个allOf条件中相同（见下面的例子）。
更复杂的条件结构可使用嵌套条件建：not，allOf或anyOf。嵌套条件可以嵌套到任意深度。

**内建的条件**

* `branch` 当正在构建的分支与给出的分支模式匹配时才执行stage，如：`when { branch 'master' }` 注：只适合多分支
* `buildingTag` 当正在构建一个Tag时执行该阶段，如：`when { buildingTag() }`
* `changelog` 当构建的SCM变更日志包含给定的正则表达式模式，则执行该阶段，如：`when { changelog '.*^\\[DEPENDENCY\\] .+$' }`
* `changeset` 当构建的SCM变更集包含一个或多个匹配给定字符串或glob的文件，则执行该阶段
* `changeRequest` 如果当前构建是针对“变更请求”(也就是GitHub和Bitbucket上的Pull请求、GitLab上的Merge请求或Gerrit中的变更等)，则执行该阶段。
* `environment` 当指定的环境变量设置为给定值时执行，如：`when { environment name: 'DEPLOY_TO', value: 'production' }`
* `equals` 当期望值等于实际值时执行该stage，如：`when { equals expected: 2, actual: currentBuild.number }`
* `expression` 当指定的Groovy表达式求值为true时执行，如：`when { expression { return params.DEBUG_BUILD } }`
* `tag` 如果TAG_NAME变量与给定的模式匹配，则执行该阶段，如: `when { tag "release-*" }`
* `not` 当嵌套条件为false时执行。必须包含一个条件，如：`when { not { branch 'master' } }`
* `allOf `当所有嵌套条件都为真时执行。必须至少包含一个条件，如：`when { allOf { branch 'master'; environment name: 'DEPLOY_TO', value: 'production' } }`
* `anyOf` 当至少一个嵌套条件为真时执行。必须至少包含一个条件，如：`when { anyOf { branch 'master'; branch 'staging' } }`
* `triggeredBy` 当给定的参数触发当前构建时，执行该阶段

> 常用的有`branch`、`expression`、`allOf`、`anyOf`

```yaml
pipeline {
    agent any
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            when {
                expression { BRANCH_NAME ==~ /(production|staging)/ }
                anyOf {
                    environment name: 'DEPLOY_TO', value: 'production'
                    environment name: 'DEPLOY_TO', value: 'staging'
                }
            }
            steps {
                echo 'Deploying'
            }
        }
    }
}
```

## 2.13 Sequential Stage

声明式 pipeline中的stage可以继续声明一个嵌套的stage列表，以便按顺序在其中运行，请注意，一个stage必须有且只有一个`steps`，`parallel`或`stages`，Sequential stage的最后一个阶段。如果`stage`指令嵌套在`parallel`块本身内，则不可能在`stage`指令内在嵌套`parallel`块。然而，`parallel`块中的`stage`指令可以使用`stage`的所有其他功能，包括`agent`, `tools`, `when`等，例如：

```yaml
pipeline {
    agent none
    stages {
        stage('Non-Sequential Stage') {
            agent {
                label 'for-non-sequential'
            }
            steps {
                echo "On Non-Sequential Stage"
            }
        }
        stage('Sequential') { # 一个stage必须有且只有一个steps，parallel 或 stages
            agent {
                label 'for-sequential'
            }
            environment {
                FOR_SEQUENTIAL = "some-value"
            }
            stages {  
                stage('In Sequential 1') {
                    steps {
                        echo "In Sequential 1"
                    }
                }
                stage('In Sequential 2') {
                    steps {
                        echo "In Sequential 2"
                    }
                }
                stage('Parallel In Sequential') {
                    parallel {  # stage指令嵌套在parallel块内，不能在stage指令内在嵌套parallel块
                        stage('In Parallel 1') {
                            steps {
                                echo "In Parallel 1"
                            }
                        }
                        stage('In Parallel 2') {
                            steps {
                                echo "In Parallel 2"
                            }
                        }
                    }
                }
            }
        }
    }
}
```

## 2.14 parallel

声明式pipeline中的stages可以在一个parallel块中声明多个嵌套的stages，在一个parallel块中，它将并行执行。请注意，一个阶段必须有且只有一个`steps`, `stages` 或`parallel` 。嵌套的stage不能包含进一个的`parallel`阶段本身，但其行为与任何其他阶段相同。

## 2.15 script

script步骤需要一个script Pipeline，并在声明式Pipeline中执行。对于大多数用例，script在声明式Pipeline中的步骤不是必须的，但它可以提供一个有用的加强。

# 3. 脚本式pipeline

Groovy脚本不一定适合所有使用者，因此jenkins创建了声明式pipeline，为编写Jenkins管道提供了一种更简单、更有主见的语法。但是不可否认，由于脚本化的pipeline是基于groovy的一种DSL语言，所以与声明式pipeline相比为jenkins用户提供了更巨大的灵活性和可扩展性。

## 3.1 流程控制

脚本化pipeline从Jenkinsfile的顶部向下连续执行，就像Groovy或其他语言中的大多数传统脚本一样。因此，提供流控制依赖于Groovy表达式，比如`if/else`条件

```yaml
node {
    stage('Example') {
        if (env.BRANCH_NAME == 'master') {
            echo 'I only execute on the master branch'
        } else {
            echo 'I execute elsewhere'
        }
    }
}
```

管理脚本化pipeline流控制的另一种方法是使用Groovy的异常处理支持。当步骤由于某种原因失败时，它们会抛出异常。错误处理行为必须使用Groovy中的`try/catch/finally`块

```yaml
node {
    stage('Example') {
        try {
            sh 'exit 1'
        }
        catch (exc) {
            echo 'Something failed, I should sound the klaxons!'
            throw
        }
    }
}
```

## 3.2 Steps

正如我们开始所讨论的，pipeline最基本的部分是“steps”。从根本上说，steps告诉Jenkins要做什么，并作为声明性和脚本化pipeline语法的基本构建块。脚本化pipeline不引入任何特定于其语法的steps;而[pipeline steps reference](https://jenkins.io/doc/pipeline/steps/)包含管道和插件提供的完整步骤列表。

# 4. Jenkins常用内置全局变量

## 4.1 为docker提供的变量

* `Image.id`：带有标签的镜像名称或者镜像ID
* `Container.id`：正在运行的容器ID
* `Container.stop`：运行docker stop和docker rm关闭容器并且删除其存储
* `Container.port(port)`： 显示容器端口和主机端口的映射方式

## 4.2 为环境变量env提供的变量

* `BRANCH_NAME`： 正在构建的分支的名称
* `BUILD_BUMBER`：当前构建号，例如“153”
* `BUILD_ID`：当前构建ID，与BUILD_NUMBER相同，但对于较旧构建，则为YYYY-MM-DD_hh-mm-ss时间戳
* `JOB_NAME`：此构建的项目名称，如“foo”或“foo/bar”
* `JOB_BASE_NAME`：此构建的项目的简短名称，将文件夹路径剥离，例如“foo”替换为“bar/foo”。
* `BUILD_TAG`：字符串“jenkins-*${JOB_NAME}*-*${BUILD_NUMBER}*"，所有前斜杠("/")都替换为斜杠("-")。
* `EXECUTOR_NUMBER`：标识执行此构建的当前执行器(同一机器的执行器之间的执行器)的惟一数字
* `NODE_NAME`：如果构建在代理上，则代理的名称;如果运行在主代理上，则为“master”
* `WORKSPACE`：分配给构建项目工作空间的目录的绝对路径。
* `NODE_LABELS`：指定节点的标签的空白分隔列表。
* `JENKINS_HOME`：在主节点上为Jenkins分配的用于存储数据的目录的绝对路径。
* `JENKINS_URL`：jenkins的完整URL，（注意：仅在系统配置中设置了Jenkins URL才可用）
* `BUILD_URL`：此构建的完整URL，如`http://server:port/jenkins/job/foo/15/`(必须设置jenkins URL)
* `JOB_URL`:此作业的完整URL，如`http://server:port/jenkins/job/foo/`(必须设置jenkins URL)

## 4.3 为currentBuild提供的变量

该`currentBuild`变量属于[RunWrapper](https://javadoc.jenkins.io/plugin/workflow-support/org/jenkinsci/plugins/workflow/support/steps/build/RunWrapper.html)类型 ，可用于引用当前运行的构建。它具有以下可读属性：

* `number`：内部编号
* `displayName`：通常`#123`但有时设置为例如SCM提交标识符。
* `result`：通常为`SUCCESS`, `UNSTABLE`, `FAILURE` （对于正在进行的构建*可能*为null）
* `currentResult`：典型地`SUCCESS`，`UNSTABLE`或`FAILURE`。永远不会为空。
* `projectName`：构建此项目的名称，例如`foo`。
* `duration`： 构建过程持续的时间 ，单位ms
* `durationString`：人们可读的构建过程持续时间
* `description`：有关构建的附加信息



