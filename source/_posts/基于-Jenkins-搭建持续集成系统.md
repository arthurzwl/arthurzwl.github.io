---
title: 基于 Jenkins 搭建持续集成系统
date: 2019-04-03 11:26:05
tags:
- jenkins
- CI/CD
categories:
- 持续集成
---

> 这是我在公司部署 Jenkins 时写的部署文档&使用手册，现在放到博客上，可以给大家一些参考，以后自己也好回顾。

## 1. 系统介绍

### 1.1 CI/CD 介绍

持续集成，Continuous integration (CI)，强调开发人员提交了新代码之后，立刻进行构建、（单元）测试。根据测试结果，我们可以确定新代码和原有代码能否正确地集成在一起。

持续交付，Continuous delivery (CD or CDE)，在持续集成的基础上，将集成后的代码部署到更贴近真实运行环境的「类生产环境」（production-like environments）中。比如，我们完成单元测试后，可以把代码部署到连接数据库的 Staging 环境中更多的测试。如果代码没有问题，可以继续手动部署到生产环境中。

持续部署，Continuous deployment (CD)，则是在持续交付的基础上，把部署到生产环境的过程自动化。

### 1.2 Jenkins 介绍

Jenkins是一个独立的开源自动化服务器，可以用来自动化与构建、测试、交付或部署软件相关的各种任务。

### 1.3 SonarQube 介绍

SonarQube，是一个开源的代码静态扫描工具。

## 2. 系统搭建

### 2.1 Jenkins 服务搭建

Jenkins 可以通过原生的系统安装包，Docker，或者任何安装了Java运行时环境( JRE )的机器都可以独立运行。

我们可以直接在官网下载 war 包，可以使用 Servlet 容器部署，也可以使用 Jenkins war 包嵌入的 Jetty 容器部署。

由于现在最新的 Jenkins 依赖 Servlet API 3.1，而我们目前使用的 Resin 实现的是 Servlet 3.0，无法部署最新的 Jenkins server。所以本着快速部署不折腾的原则，我这里使用的是 war 包嵌入的 Jetty 容器部署的。运行命令如下：

```shell
java -jar jenkins.war --httpPort=8080
```

### 2.2 SonarQube 服务搭建

SonarQube 使用了 Docker 部署。

## 3. 服务整合

> 服务目前已经都配置好，不需要重复配置，这里列出详细步骤是方便大家了解服务整合方法，提供一些参考。

### 3.1 Gitlab & Jenkins 服务整合

Gitlab 和 Jenkins 服务整合，主要是需要当开发同学 push 代码，或者提交 merge request 时可以自动触发 Jenkins 持续集成任务，而非手动运行持续集成任务。

配置步骤：

1. Jenkins 需要安装 gitlab-plugin 插件
2. 在 Mange Jenkins -\> Configure System -\> Gitlab 添加 Gitlab 配置
3. 在 Gitlab 里对应项目给 robot 用户赋予 Developer 权限
4. 在 Gitlab 里对应项目添加 webhook，添加步骤参考下文

### 3.2 Jenkins & SonarQube 服务整合

Jenkins 和 SonarQube 服务整合的目的，是在 Pipeline 中可以调用 SonarQube Server 对代码进行静态扫描，并且在扫描结束后可以获取扫描结果

配置步骤：

1. Jenkins 安装 sonar 插件
2. Sonar 登录 -\> 点击头像 -\> My Account -\> Security -\> Generate Tokens，这里保存生成的 token，后续会使用到
3. Jenkins, 选择 Manage Jenkins -\> Configure System -\> SonarQube servers, 添加 SonarQube Server 配置，填写上一步生成的 token
4. Sonar, 选择 Administration -\> Configuration -\> Webhooks 添加 Jenkins webhook

## 4. 如何使用

### 4.1 Jenkins 使用介绍

#### 4.1.1 新建任务

目前我搭建好的 Jenkins 支持新建几种任务

1. Freestyle project
2. Maven project
3. Pipeline
4. Multi-configuration project
5. Multibranch Pipeline

我目前推荐使用的是 Pipeline。新建任务步骤如下：

1. 点解 New Item
2. 填写任务名（注意任务名不能重复，最好以项目名开头，后续可能会根据项目名分配权限），选择任务类型 Pipeline
3. GitLab Connection 选择对应配置
4. 选择 Build Triggers，勾选 Build when a change is pushed to GitLab（需要在 gitlab 里添加 webhook，后续会说），勾选需要的 event，这里需要保存 GitLab webhook URL，后续会用到
5. 填写 Pipeline，这里支持两种方式，可以根据需要选择，6、7中选择一种
6. Pipeline script，直接在Jenkins 上写 Pipeline definition
7. Pipelione script from SCM，从代码仓库里下载 Jenkinsfile
8. 在 Gitlab 里对应项目给 robot 用户赋予 Developer 权限，也可以直接在 Gitlab Group 里赋予 robot 用户赋予 Developer 权限
9. 在GitLab里配置 webhook，这里 GitLab 配置 webhook 需要 master 权限打开项目点击 Settings，选择 Integrations
10. 勾选需要的Trigger，填写 URL，点击 Add webhook，填写第4步得到的 GitLab webhook URL，注意这里需要在url上添加认证信息，类似 [http://user:token@localhost:8080/project/project-name]，建议使用 robot 的 token

#### 4.1.2 Pipeline 介绍

Jenkins Pipeline 是一组插件，支持在 Jenkins 里实现和集成持续交付。

简而言之，就是一套运行于Jenkins上的工作流框架，将原本独立运行于单个或者多个节点的任务连接起来，实现单个任务难以完成的复杂流程编排与可视化。

Pipeline 提供了一套可扩展的工具，通过  Pipeline 领域特定语言 (DSL) syntax 可以实现 Pipelines  as code。

#### 4.1.3 Jenkinsfile 语法介绍

> 待完善

可参考：

1. Pipeline Syntax, [https://jenkins.io/doc/book/pipeline/syntax/]
2. abayer, Jenkinsfile, In GitHubGist, [https://gist.github.com/abayer/925c68132b67254147efd8b86255fd76]

#### 4.1.4 任务结果通知

目前任务结果通知可以支持邮件和Gitlab comment 的方式
Pipeline 配置方式

post {
  always {
    echo 'send email'

      script {
        env.committeremail = sh (
            script: 'git --no-pager show -s --format=\'%ae\'',
            returnstdout: true
            ).trim()
      }

    emailext (
        subject: "[Jenkins] ${currentbuild.currentresult}: job ${env.job_name} #${env.build_number}",
        body: """
        check console output at <a href="${env.build_url}">${env.job_name} #${env.build_number}</a>""",
        to: "${env.committeremail}",
        )

      script { // gitlab comment
        def resulticon = currentbuild.result == 'success' ? ':white_check_mark:' : ':anguished:'
          addgitlabmrcomment comment: "$resulticon jenkins build $currentbuild.result\n\nresults available at: [jenkins [$env.job_name#$env.build_number]]($env.build_url)"
      }
  }
}

#### 4.1.5 权限管理

Jenkins 目前已经对接了公司的单点登录，权限是基于角色进行管理的。在 Manage Jenkins -\> Manage and Assign Roles 里进行管理。

#### 4.1.5 一些 Jenkins 插件的使用介绍

> 待完善

##### 4.1.5.1 如何安装插件

Jenkins 安装插件有多种方式。

1. 在 Manage Jenkins -\> Plugin Manager 里直接安装
2. 从网站将插件下载下来，放到 $JENKINS\_HOME/plugins 目录下，重启 Jenkins 服务

因为内网的服务器无法正常连接 Jenkins 的插件更新网站，且为了能够方便部署，在服务器有问题时，能够快速恢复服务，切换服务器，我这里选择了第二种方式。

这里推荐我之前写的脚本，[https://github.com/arthurzwl/jenkins-ci-plugins-installer]，可以方便批量下载插件以及插件的依赖，里面已经列举了常用插件，可以直接使用

### 4.2 SonarQube 使用介绍

> 待完善

## 5. 参考链接

* 如何理解持续集成、持续交付、持续部署？ - yumminhuang的回答 - 知乎 [https://www.zhihu.com/question/23444990/answer/89426003]
* Jenkins User Documentation, [https://jenkins.io/doc/#jenkins-user-documentation]
* SonarQube Documentation, [https://docs.sonarqube.org/7.7/]
* SonarQube. In Wikipedia, [https://en.wikipedia.org/wiki/SonarQube]
