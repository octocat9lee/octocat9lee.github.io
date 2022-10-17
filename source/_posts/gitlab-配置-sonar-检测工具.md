---
title: gitlab 配置 sonar 检测工具
tags:
  - 技术点滴
toc: true
date: 2022-10-17 10:04:16
---

本文主要介绍在 `gitlab` 管理仓库中配置 `sonar` 代码质量检查工具。

# 操作步骤
``` bash
1 必须首先配置 master 分支，其他分支根据实际情况进行配置
2 在仓库配置完成后，通过 [CI/CI]-> [流水线] 菜单进行查看
```

<!--more-->

# .gitlab-ci.yml 配置

``` bash
新增文件名: .gitlab-ci.yml  # 注意文件名使用 . 开头

文件内容：

stages:
  - sonarqube_scan
  - sendmail

sonarqube_scan_job:
  stage: sonarqube_scan
  script:
    - sonar-scanner -Dsonar.projectName=$CI_PROJECT_NAME -Dsonar.projectKey=$CI_PROJECT_NAME -Dsonar.branch.name=${CI_COMMIT_REF_NAME}
  tags:
    - sonar-scanner
  when: always

sendmail_job:
  stage: sendmail
  script:
    - echo $GITLAB_USER_EMAIL
    - echo $CI_PROJECT_NAME
    - echo $CI_COMMIT_REF_NAME
    - python /opt/sonarqube_api.py $CI_PROJECT_NAME $CI_COMMIT_REF_NAME $GITLAB_USER_EMAIL
    
  tags:
    - sonar-scanner
```

# sonar-project.properties 配置

## Java 项目配置
``` bash
新增文件名：sonar-project.properties

文件内容：

sonar.projectKey=keeperplus
sonar.projectName=keeperplus
sonar.projectVersion=1.0
sonar.sources=.
sonar.sourceEncoding=UTF-8
sonar.language=java
sonar.java.source=1.8
sonar.java.binaries=.

注意：需根据自己项目修改属性 sonar.projectKey、sonar.projectName
```

## NodeJS 项目配置
``` bash
新增文件名：sonar-project.properties

文件内容：

sonar.projectKey=test-ui
sonar.projectName=test-ui
sonar.projectVersion=1.0
sonar.sourceEncoding=UTF-8
sonar.modules=javascript-module
javascript-module.sonar.projectName=test-ui
javascript-module.sonar.language=js
javascript-module.sonar.sources=.
javascript-module.sonar.exclusions=**/lib/**/*,**/node_modules/**/*,**/public/**/*
javascript-module.sonar.projectBaseDir=.

#注意：需根据自己项目修改属性 sonar.projectKey、sonar.projectName、javascriptmodule.sonar.projectName
```

## C++ 项目配置
``` bash
新增文件名：sonar-project.properties

文件内容：
sonar.projectKey=test
sonar.projectName=test
sonar.projectVersion=1.0
sonar.sources=.
sonar.sourceEncoding=UTF-8
sonar.language=c++
sonar.cxx.file.suffixes=.cxx,.cpp,.cc,.c,.hxx,.hh,.handled

#注意：需根据自己项目修改属性 sonar.projectKey、sonar.projectN
```
