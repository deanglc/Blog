---
title: "Docker镜像瘦身&Go mod初体验"
description: ""
tags: [ "golang","docker" ]
categories: [ "golang", "docker"]
keywords: [ "golang", "docker" ]
isCJKLanguage: true

date: 2019-03-19T11:00:53
draft: false
---

##Docker镜像瘦身&Go mod初体验

​    go1.11版本正式上线了 go module,研究了一哈,此次示例用上.

```go
Useage:	
	go mod <command> [arguments]

The commands are:

	download    download modules to local cache //ci/cd使用
	edit        edit go.mod from tools or scripts
	graph       print module requirement graph
	init        initialize new module in current directory
	tidy        add missing and remove unused modules
	vendor      make vendored copy of dependencies
	verify      verify dependencies have expected content
	why         explain why packages or modules are needed
```



### Dockerfile变化

**原dockerfile**

```dockerfile
FROM golang:alpine as builder
COPY src/ /opt/src
COPY Makefile /opt
WORKDIR /opt
RUN apk add --update make upx git
RUN go get -u github.com/gin-gonic/gin \
    && go get -u github.com/gin-contrib/cors \
    && go get -u github.com/sirupsen/logrus \
    && go get -u github.com/fatih/color \
    && go get -u github.com/spf13/cobra \
    && go get -u github.com/go-sql-driver/mysql \
    && go get -u github.com/jinzhu/gorm \
    && go get -u github.com/dgrijalva/jwt-go \
    && go get -u github.com/bitly/go-simplejson \
    && go get -u github.com/lestrrat/go-file-rotatelogs \
    && go get -u github.com/getsentry/raven-go \
    && go get -u github.com/streadway/amqp \
    && go get -u github.com/tidwall/gjson \
    && go get -u github.com/spf13/viper \
    && go get -u github.com/rifflock/lfshook
RUN make docker
# 以上为build
FROM alpine
COPY --from=builder /opt/robotsln /usr/local/bin/
COPY --from=builder /opt/conf.yaml /src/robocli/
# Refer: http://blog.cloud66.com/x509-error-when-using-https-inside-a-docker-container/
RUN apk add --no-cache --update ca-certificates tzdata
ENV TZ Asia/Shanghai
EXPOSE 9008
CMD [ "project", "-l", ":9008"]
```

**使用go mod的新Dockerfile**

```dockerfile
FROM golang:1.12 as build

ENV GOPROXY https://goproxy.cn
ENV GO111MODULE on

WORKDIR /go/cache

COPY go.mod .
COPY go.sum .
RUN go mod download

WORKDIR /go/release

ADD . .

RUN GOOS=linux CGO_ENABLED=0 go build -ldflags="-s -w" -installsuffix cgo -o app main.go

FROM scratch as prod # 

COPY --from=build /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
COPY --from=build /go/release/app /
COPY --from=build /go/release/conf.yaml /

CMD ["/app"]

```

这个项目有一些外部依赖，在本地开发的时候都已调整好，并且编译通过，在本地开发环境已经生成了两个文件`go.mod`、`go.sum`

在dockerfile的第一步骤中，先启动module模式，且配置代理.(如进行CI\CD的服务器在香港等地可以用设置**GOPROXY**)

指令`RUN go mod download`执行的时候，会构建一层缓存，包含了该项所有的依赖。之后再次提交的代码中，若是`go.mod`、`go.sum`没有变化，就会直接使用该缓存，起到加速构建的作用，也`不用重复的去外网下载依赖`了。若是这两个文件发生了变化，就会重新构建这个缓存分层。

#### go构建命令使用`-ldflags="-s -w"`

在官方文档：[Command_Line](https://link.juejin.im/?target=https%3A%2F%2Fgolang.org%2Fcmd%2Flink%2F%23hdr-Command_Line)里面说名了`-s -w`参数的意义，按需选择即可。

- `-s`: 省略符号表和调试信息
- `-w`: 省略DWARF符号表

#### 使用scratch镜像

使用`golang:1.12`开发镜像构建好应用后，在使用`scratch`来包裹生成二进制程序。

关于`最小基础镜像`，docker里面有这几类：

- scratch: 空的基础镜像，最小的基础镜像
- busybox: 带一些常用的工具，方便调试， 以及它的一些扩展busybox:glibc
- alpine: 另一个常用的基础镜像，带包管理功能，方便下载其它依赖的包



```dockerfile
FROM golang:alpine as builder  
COPY src/ /opt/src                                                                                                                                    COPY Makefile /opt                                                                                                                                            WORKDIR /opt                                                                                                                                         RUN apk add --update make upx git                                                                                                   RUN go get -u github.com/gin-gonic/gin \\                                                                                                      && go get -u github.com/gin-contrib/cors \                                                                                                       && go get -u github.com/sirupsen/logrus \                                                                                                           && go get -u github.com/fatih/color \                                                                                                           && go get -u github.com/spf13/cobra \                                                                                                   && go get -u github.com/go-sql-driver/mysql \                                                                                                           && go get -u github.com/jinzhu/gorm \                                                                                                      && go get -u github.com/dgrijalva/jwt-go \                                                                                                   && go get -u github.com/bitly/go-simplejson \                                                                                           && go get -u github.com/lestrrat/go-file-rotatelogs \                                                                                                    && go get -u github.com/getsentry/raven-go \                                                                                                        && go get -u github.com/streadway/amqp \                                                                                                         && go get -u github.com/tidwall/gjson \                                                                                                           && go get -u github.com/spf13/viper \                                                                                                                                                && go get -u github.com/rifflock/lfshook
                                                                                                                                            RUN make docker
                                                                                                                                            FROM scratch                                                                                                                                COPY --from=builder /opt/robotsln /usr/local/bin/
COPY --from=builder /opt/conf.yaml /src/robocli/
RUN ls
                                                                                                                                            # Refer: http://blog.cloud66.com/x509-error-when-using-https-inside-a-docker-container/
                                                                                                                                            RUN apk add --no-cache --update ca-certificates tzdata
                                                                                                                                            ENV TZ Asia/Shanghai
                                                                                                                                           EXPOSE 9008
                                                                                                                                            CMD [ "project", "-l", ":9008","-d","database.connection"]
```

```dockerfile
# 1.go-build   这一部分使用go mod download
FROM golang:alpine AS build
 # 设置我们应用程序的工作目录
WORKDIR /go/src/github.com/path/to/project
# 添加所有需要编译的应用代码
ADD . .
# 编译一个静态的go应用（在二进制构建中包含C语言依赖库）
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo .
 # 设置我们应用程序的启动命令
CMD ["./blog-multistage-go"]

# 2.CERTS Stage
FROM alpine:latest as certs
# Install the CA certificates
#RUN apk --update add tzdata
RUN apk add --no-cache --update ca-certificates 


# 3.PRODUCTION STAGE
FROM scratch
# 从certs阶段拷贝CA证书

COPY --from=certs /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
# 从buil阶段拷贝二进制文件
COPY --from=build /go/src/github.com/path/to/project .
# 时区
COPY --from=certs /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
CMD ["./project"]

```

提交代码触发了CI/CD,上服务器看了看新构建的容器比之前小了5MB.



####

####ADD 还是 COPY?

copy是add的精简版.在大部分情况下docker官方推荐使用copy.

add可以获取https://dddd.com/test.go 这样的文件到本地,甚至可以压缩.因为历史原因,才新增了精简版——copy.

