---
layout: post
title: Depmod CI流程下Go module依赖最低版本的控制
category: 工作
tags: 
  - go
  - dependency-manager
typora-root-url: ..
---


### 背景

延续之前的 [Go微服务之间模块依赖管理的思考]({% link _posts/2021-07-18-Go微服务之间模块依赖管理的思考.md %}) 最后的想法，**干预CI流程管控服务版本** 为思路我实现了一个go项目 [Depmod](https://github.com/tangtj/depmod) 。


### 介绍
解析指定的 go.mod 文件，分析项目所使用的第三方依赖并与给的配置做对比。出现违背配置的依赖时，非 0 返回码退出影响CI的通过。

#### depmod
使用了 `golang.org/x/mod`库，可以直接解析go.mod文件提取依赖的信息。代码太简单了，没啥可以说的。depmod在不符合预期时将会非0返回码退出，这将会中断流程，阻止编译成功。逼迫团队内其他成员升级/移除依赖

#### config
依赖的配置我选择使用外置的配置文件，我在gitlab上新建了一个repo专门给用来放置配置信息，这样配置的管理是可控的，在内网也方便使用。
```json
{
  "mods":[
    {
      "name":"golang.org/x/mod", //依赖的路径
      "min_ver":"v0.4.3", // 最低可使用版本号
      "deprecated": true //是否弃用
    }
  ],
  "deprecates": [ //弃用依赖的列表，在列表中的无法使用，将会直接报错
    "github.com/go-redis/redis/v7"
  ]
}
```

### 使用
提供一个github-action的简单配置。
```
name: depmod github action 流程测试
on: [push]
jobs:
  check-mod-version:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-go@v2
        with:
          go-version: '^1.16' # The Go version to download (if necessary) and use.
      - run: go get github.com/tangtj/depmod@master
      - run: go install github.com/tangtj/depmod@master
      # config.json 可以从其他地方弄过来
      - run: depmod -c config-pass.json -m go.mod
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-go@v2
      - run: go build cmd/main.go
```