# Jenkins + Groovy学习笔记

![](https://mmbiz.qpic.cn/mmbiz_png/CIfGokpaahe9STPdicnu5w67qMY1vRkfx8nYsSibdSlTV1XU96Jf9X9fO19LbtdS5otBhElxBnicNvarOeCz6DG2w/0?wx_fmt=png)

之前使用 `Jenkins` + `Bash` 的方式实现公司项目的CI/CD，感觉还好，工作也很顺畅，没什么大毛病，只是多了对脚本的修修改改。最近看到很多朋友（或网上）在讲 `pipeline` 。好奇心重的我，也去查了查资料，学习了下。本文是对学习 `pipeline` 的一个实践及总结。

## 什么是CI/CD

```text
CI - Continue Integrate         # 持续集成
CD - Continue Deploy/Delever    # 持续部署/交付
```

详细的解释如下[CI/CD](https://www.jianshu.com/p/5643b1cf9e3f "什么是 CI/CD?")

### 持续集成（CI）

通过持续集成，开发人员能够频繁地将其代码集成到公共代码仓库的主分支中。 开发人员能够在任何时候多次向仓库提交作品，而不是独立地开发每个功能模块并在开发周期结束时一一提交。

### 持续部署/持续交付（CD）

持续部署扩展了持续交付，以便软件构建在通过所有测试时自动部署。在这样的流程中， 不需要人为决定何时及如何投入生产环境。CI/CD 系统的最后一步将在构建后的组件/包退出流水线时自动部署。 此类自动部署可以配置为快速向客户分发组件、功能模块或修复补丁，并准确说明当前提供的内容。

持续集成（CI）注重将各个开发者的工作集合到一个代码仓库中，通常每天会进行几次， 主要目的是尽早发现集成错误，使团队更加紧密结合，更好地协作。 持续交付/持续部署是一种更高程度的自动化，无论何时代码有较大改动， 都会自动进行构建／部署。

了解了什么是CI/CD后，我们看看CI/CD落地情况。日常工作中，大多数公司选择 `Jenkins` 作为CI/CD落地的重要工具，我们公司也不例外。下面开始我们激动人心的实践吧，我都快等不及了└(^o^)┘。

## 使用pipeline实战

开始之前，先了解下 `pipeline` 的一些特性及语法

### pipeline功能特点

- 是帮助jenkins实现持续集成（CI）转变为持续部署（CD）的重要功能插件；
- 将多个节点的单个任务连接起来，实现单个任务难以实现的复杂发布流程；
- Pipeline 的实现方式是一套 Groovy DSL，所有的发布流程都可以表述为一段 Groovy 脚本；
- 是jenkins上的一套工作流框架。

### pipeline语法

- stage：pipeline可以划分为多个stage阶段，每个是stage为执行的一个操作，每个阶段可以跨节点；
- node：jenkins的节点，是执行操作的具体服务器；
- step：是jenkins pipeline执行操作的最小单元。

### 实战

通过创建一个`Pipeline`项目，完成项目编译，发布过程，并实现告警提示。前置条件如下：

```text
项目类型：spring boot，gradle打包
项目地址：https://github.com/slient2020/spring-boot-example.git
项目分支：develop
其他：企业微信
```

如何在Jenkins上创建pipeline项目，参考[](https://www.geek-share.com/detail/2775135464.html  "jenkins的pipeline实现指定节点项目构建并部署代码至后端服务器")

#### Pipeline 中groovy脚本编写

在Jenkins中创建一个`Pipeline`类型的项目，在`Pipeline`脚本编辑区域，填写如下内容。

```groovy
// 导入工具类，计算过程执行时间
import hudson.Util
// 定义常量及变量
// 项目地址
// 外部定义的变量，不能通过${}方式在脚本中应用
String gitURL = 'https://github.com/slient2020/spring-boot-example.git'
// 拉取的项目分支
String gitBranch = 'develop'
// 定义变量scmVars用于获取git相关信息，如commit_hash
def scmVars
// 定义项目名称变量
def projectName
// 定义一个过程执行时间
def duration

// 消除jenkins 日志上调试shell日志
def turnOffShDebug(cmd, returnStatus) {
    return sh (script: '#!/bin/sh -e\n'+ cmd, returnStatus: returnStatus)
}

/*
 发送消息到企业微信
 参数说明
 key 企业微信token
 msg 需要发送的消息内容，string类型
 metionLists 艾特人手机号，逗号分割
*/

def sendMsg(String qywxkey, String msg, String metionsLists) {
    turnOffShDebug ('''
    #!/bin/sh -e
    curl -s 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=''' + qywxkey + '''' \
    -H 'Content-Type: application/json' \
    -d ' { "msgtype": "text", "text": { "content": "'''+ msg +'''", "mentioned_mobile_list":["'''+ metionsLists +'''"] } }'
    ''', true)
}

// 入口
pipeline {
    agent any // 可以运行在任意节点
    environment {
        // 拉取github代码的凭证，配置在Jenkins上
        gitCredentialsId = '70af0800-2fb3-xxx-xxxx-xxxxxxxx'
        // 企业微信接收消息的key
        key = '138xxxx-xxxx-xxxx-xxxx-xxxxxxxxx'
        // 打包过程失败@通知人
        user = '158185xxxxx'
    }

    tools {
        // 定义gradle版本，与jenkins的Global Tool Configuration中配置一致
        gradle 'gradle 5.4'
    }

    // 自动同步拉取代码频度
    triggers {
        // 每8小时自动打一次包
        cron('H */8 * * *')
        // 每5分钟拉取代码，检查变动，决定是否打包
        pollSCM('H/5 * * * *')
    }

    stages {
        // 步骤一：获取仓库代码
        stage('SCM') {
            steps {
                git branch: gitBranch, credentialsId: gitCredentialsId , url: gitURL
                script{
                    scmVars = git branch: gitBranch, credentialsId: gitCredentialsId, url: gitURL
                    // 此处的commitHash为全局变量
                    commitHash = scmVars.GIT_COMMIT
                    // 获取commit_hash的前10个字符串
                    docker_tag = commitHash.substring(0,10)
                }
                script {
                    // env.GIT_COMMIT_MSG 为全局变量
                    env.GIT_COMMIT_MSG = sh (script: 'git log -1 --pretty=%B ${GIT_COMMIT}', returnStdout: true).trim()
                    // 从给定的gitURL中获取项目名称，设定project为全局变量
                    projectName = sh (script: 'echo '+ gitURL + ' | xargs -I{} basename {} | sed "s/.git//"', returnStdout: true).trim() as String
                    // 获取项目版本信息
                    projectVersion=sh (script: 'cat ${WORKSPACE}/build_scripts/application_version.gradle | grep version | awk \'{print $3}\' | sed "s/\'//g"', returnStdout: true).trim() as String
                }
            }
            // 判断本步骤是否成功，不成功则发送消息提示
            post{
                success {
                    // 成功，不做任何操作，继续后面的操作
                    println('continue...')
                }
                failure { // 本步骤失败，退出后续操作，并发送消息
                    script {
                        String msg = projectName + '项目的分支'+ gitBranch +',hash:'+commitHash+'拉取代码失败，请检查网络等，后续操作已终止！'
                        sendMsg(key, msg, user)
                        error "Failed, exiting now..."
                    }
                }
                aborted { // 本步骤被终止
                    echo "aborted"
                }
                unstable { // 不稳定，并退出
                    script{
                        error "Unstable, exiting now..."
                    }
                }
            }
        }

        // 步骤二：编译代码
        stage('Compile!') {
            // 检查是否自动打包
            when {
                expression {
                    !skipRemainingStages
                }
            }
            steps {
                sh 'gradle --version'
                sh 'gradle clean build -x integrationTest'
            }
            // 判断本步骤是否成功，不成功则发送消息提示
            post{
                success {
                    // 成功，不做任何操作，继续后面的操作
                    println('continue...')
                }
                failure { // 本步骤失败，退出后续操作，并发送消息
                    script {
                        // 将日志拷贝到远端web服务器，并获取到日志访问地址
                        //    todo: 读取jenkins的打包日志获取报告内容
                        String msg = '【'+ projectName +'打包失败】\n  - 分支：'+ gitBranch +'\n  - commit_Hash: '+commitHash
                        // 发送给运维
                        sendMsg(key, msg, user)
                        error "Failed, exiting now..."
                    }
                }
                aborted { // 本步骤被终止
                    echo "aborted"
                }
                unstable { // 不稳定，并退出
                    script{
                        error "Unstable, exiting now..."
                    }
                }
            }
        }

        // 步骤三：更新应用
        stage('upload!'){
            steps {
                script {
                    sh '''
                    bash update.sh
                    '''
                }
            }
        }
    }

    // 最后执行
    post {
        success {
            script{
                duration = Util.getTimeSpanString(System.currentTimeMillis() - currentBuild.startTimeInMillis)
                String msg = '【'+ projectName +'打包成功】\n  - 分支：' + gitBranch + '\n  - commit_Hash: ' + commitHash + '\n  - 总用时: ' + duration +'\n。部署成功！'
                sendMsg(key, msg, user)
            }
        }
    }
}
```

## 总结

对比`Jenkins` + `Bash`的方式，显然`Pipeline`方式更受运维人员喜爱，毕竟简单，脚本也容易编写。但一般公司中，开发人员喜欢控制自己代码CI/CD，`Pipeline`这种方式也颇受欢迎。
