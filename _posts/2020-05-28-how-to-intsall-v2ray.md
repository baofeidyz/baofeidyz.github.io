---
layout: post
title: how to install v2ray
date: 2020-05-28
Author: baofeidyz
categories: 科学上网
tags: [科学上网]
comments: true
---

## For Linux

First, run the command blow as root

```shell
bash <(curl -L -s https://install.direct/go.sh)
```

Second, use [get v2ray-config file online](https://htfy96.github.io/v2ray-config-gen/) to create your own `config.json` file

> `config.json` file path : `/etc/v2ray/config.json`

Third, run the command to start v2ray as below

```shell
v2ray --config=/etc/v2ray/config.json
```

> note: if you get error message as below
>
> ```shell
> bash: v2ray command not found
> ```
>
> you can use `find` command to find the v2ray file path as below
>
> ```shell
> find / -name v2ray
> ```
>
> maybe you can use
>
> ```shell
> /usr/bin/v2ray/v2ray --config=/etc/v2ray/config.json
> ```
>
> btw, you can use below command to start v2ray as service
>
> ```shell
> sudo systemctl start v2ray
> ```

## For Windows

[V2RayW-github](https://github.com/Cenmrev/V2RayW)

## Others

[v2ray-home](https://www.v2ray.com)

[v2ray-github](https://github.com/v2ray/v2ray-core)

[v2ray-tools](https://www.v2ray.com/awesome/tools.html)