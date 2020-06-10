---
layout: post
title: gitlab runner的安装与注册
date: 2020-05-28
Author: baofeidyz
categories: gitlabci
tags: [gitlabci]
comments: true
---


## 1.访问项目的gitlab页面获取token

> `Settings` -> `CI/CD` -> `Runners`


获取到当前项目的key，如下图所示：

![](https://raw.githubusercontent.com/baofeidyz/images/master/img/20200529000838.png)

此时我们就可以拿到对应的token了

## 2. 安装gitlab runner
> 官网安装教程：https://docs.gitlab.com/runner/install/

建议是安装到CentOS服务器上

1. 添加源
```shell
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
```
2. 通过包管理工具安装
```shell
sudo yum install gitlab-runner
```
默认会安装最新的版本，如果需要安装特定版本则可以通过以下命令实现
```shell
yum list gitlab-runner --showduplicates | sort -r
sudo yum install gitlab-runner-10.0.0-1
```
## 3. 注册gitlab runner
```shell
sudo gitlab-runner register
```
需要输入项目所在的gitlab地址，类似于`https://gitlab.xxx.com`
回车过后需要输入项目的token，也就是我们在第一步中获取到的
后面会要求输入一些描述，紧接着是`tag`，这个可以用于`.gitlab-ci.yml`文件中指定具体的`runner`，可以认为`tag`就是这个当前注册runner的身份ID。最后是选择执行器，初级选手，只会使用`shell`,其他相关的请看[EXECUTORS执行器简单介绍](#EXECUTORS执行器简单介绍)

另，一台服务器可以注册多个`gitlab runner`

# EXECUTORS执行器简单介绍

| 执行者             | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| shell              | 在本地运行构建，默认                                         |
| docker             | 使用Docker容器运行构建 - 这需要在Runner运行的系统上安装[runners.docker]和Docker Engine |
| docker-ssh         | 运行生成使用泊坞容器，但用SSH连接到它-这需要的存在[runners.docker]，[runners.ssh]以及码头工人引擎的亚军运行的系统上安装。注意：这将在本地计算机上运行docker容器，它只是更改命令在该容器内运行的方式。如果要在外部计算机上运行docker命令，则应更改host该runners.docker部分中的参数。 |
| ssh                | 使用SSH远程运行构建 - 这需要存在 [runners.ssh]               |
| parallels          | 使用Parallels VM运行构建，但使用SSH连接到它 - 这需要存在[runners.parallels]和[runners.ssh] |
| virtualbox         | 使用VirtualBox VM运行构建，但使用SSH连接到它 - 这需要存在[runners.virtualbox]和[runners.ssh] |
| docker+machine     | 喜欢docker，但使用自动缩放的Docker机器 - 这需要存在[runners.docker]和[runners.machine] |
| docker-ssh+machine | 喜欢docker-ssh，但使用自动缩放的Docker机器 - 这需要存在[runners.docker]和[runners.machine] |
| kubernetes         | 使用Kubernetes Pods运行构建 - 这需要存在 [runners.kubernetes] |

## shell执行器补充
shell执行器是根据当前gitlab runner所安装的操作系统来决定的

| shell      | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| bash       | 生成Bash（Bourne-shell）脚本。在Bash上下文中执行的所有命令（所有Unix系统的默认值） |
| sh         | 生成Sh（Bourne-shell）脚本。Sh上下文中执行的所有命令（适用bash于所有Unix系统的后备） |
| cmd        | 生成Windows批处理脚本。所有命令都在批处理上下文中执行（Windows的默认值） |
| powershell | 生成Windows PowerShell脚本。所有命令都在PowerShell上下文中执行 |


# .gitlab-ci.yml基本语法
以下内容暂时只针对shell执行器

# config.toml基本配置
> 官方文档介绍：https://docs.gitlab.com/runner/configuration/advanced-configuration.html

## 全局配置

| 关键字           | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| `concurrent`     | 限制全局可以同时运行的作业数。使用所有已定义的运行者的作业的最大上限。0并不是指无限制，如果服务器性能还不错，可以尝试给个5 |
| `log_level`      | 日志级别（选项：调试，信息，警告，错误，致命，恐慌）。请注意，此设置的优先级低于命令行参数设置的级别`--debug`，`-l`或`--log-level` |
| `log_format`     | 日志格式（选项：runner，text，json）。请注意，此设置的优先级低于命令行参数设置的格式`--log-format` |
| `check_interval` | 定义新作业检查之间的间隔长度（以秒为单位）。默认值为3; 如果设置为0或更低，将使用默认值。 |
| `sentry_dsn`     | 启用跟踪哨兵的所有系统级错误                                 |
| `listen_address` | 地址（`<host>:<port>`），Prometheus指标HTTP服务器应该在其上监听 |

## `[session_server]`介绍

此项配置应在`[[runners]]`外部指定，主要包含以下参数

| 设置                | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| `listen_address`    | 用于会话服务器的内部URL。                                    |
| `advertise_address` | Runner将向GitLab公开的URL，用于访问会话服务器。`listen_address`如果没有定义，则回退到。 |
| `session_timeout`   | 作业完成后会话可以保持活动状态的多长时间（这将阻止作业完成），默认为1800（30分钟）。 |

> 其中`listen_address`和`advertise_address`需要以`host:port`形式提供，其中`host`可以是IP地址，也可以是域名。

示例：
```toml
[session_server]
  listen_address = "0.0.0.0:8093" #  listen on all available interfaces on port 8093
  advertise_address = "runner-host-name.tld:8093"
  session_timeout = 1800
```
## `[[runners]]`介绍

一个`config.toml`文件允许存在多个`[[runners]]`，也就对应了上文中提到的一个服务器允许注册多个gitlab runner
相关配置参数介绍如下表所示：

| 设置                   | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| `name`                 | Runner的描述，只是提供信息                                   |
| `url`                  | GitLab URL                                                   |
| `token`                | Runner的特殊令牌（不要与注册令牌混淆）                       |
| `tls-ca-file`          | 包含证书的文件，用于在使用HTTPS时验证对等方                  |
| `tls-cert-file`        | 包含证书的文件，以便在使用HTTPS时与对等方进行身份验证        |
| `tls-key-file`         | 包含私钥的文件，用于在使用HTTPS时与对等方进行身份验证        |
| `limit`                | 限制此令牌可同时处理的作业数。0（默认）仅表示不限制          |
| `executor`             | 选择应如何构建项目                                           |
| `shell`                | 用于生成脚本的shell的名称。默认值取决于平台。                |
| `builds_dir`           | 构建将存储在所选执行程序的上下文中的目录（本地，Docker，SSH） |
| `cache_dir`            | 构建缓存的目录将存储在所选执行程序（本地，Docker，SSH）的上下文中。如果使用docker执行程序，则此目录需要包含在其volumes参数中。 |
| `environment`          | 附加或覆盖环境变量                                           |
| `request_concurrency`  | 限制GitLab新作业的并发请求数（默认值为1）                    |
| `output_limit`         | 设置最大构建日志大小（以KB为单位），默认设置为4096（4MB）    |
| `pre_clone_script`     | 在克隆Git存储库之前要在Runner上执行的命令。例如，这可以用于首先调整Git客户端配置。要插入多个命令，请使用（三引号）多行字符串或“\ n”字符。 |
| `pre_build_script`     | 在克隆Git存储库之后但在执行构建之前要在Runner上执行的命令。要插入多个命令，请使用（三引号）多行字符串或“\ n”字符。 |
| `post_build_script`    | 在执行构建之后但在执行之前要在Runner上执行的命令after_script。要插入多个命令，请使用（三引号）多行字符串或“\ n”字符。 |
| `clone_url`            | 覆盖GitLab实例的URL。如果Runner无法连接到GitLab上的GitLab暴露自己，则使用。 |
| `debug_trace_disabled` | 禁用该CI_DEBUG_TRACE功能。设置为true时，即使用户CI_DEBUG_TRACE将设置为调试跟踪，也将保持禁用状态true。 |

示例：
```toml
[[runners]]
  name = "ruby-2.1-docker"
  url = "https://CI/"
  token = "TOKEN"
  limit = 0
  executor = "docker"
  builds_dir = ""
  shell = ""
  environment = ["ENV=value", "LC_ALL=en_US.UTF-8"]
  clone_url = "http://gitlab.example.local"
```