---
layout:     post
title:      docker-compose 服务启动依赖depends_on不够满足？
subtitle:   docker-compose 启动顺序控制
date:       2023-05-10
author:     kuba
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - docker
    - docker-compose
    - depends_on
---

# docker-compose 服务启动依赖depends_on不够满足？

--
docker-compose 用于构建本地开发环境是十分方便的，开发时使用docker-compose本地启动一组服务，然而对于一些特定到服务，简单到depends_on编排并不能貌似解决不了启动先后顺序问题，为啥呢？  
（如何解决服务相互依赖，以及严格的启动顺序）


---

## Install Docker
>[docker-compose](https://docs.docker.com/compose/) is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration.   

## 简单的docker-compose.yaml例子：
```yaml
version: '2'

services:
  postgres:
    image: postgres
    container_name: my_postgres
    restart: unless-stopped
    command:
      - 'postgres'
      - '-c'
      - 'max_connections=100'
      - '-c'
      - 'shared_buffers=256MB'
    environment:
      POSTGRES_DB: my_db1
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpassword
    ports:
      - 5432:5432
    volumes:
      - pg-data:/data/postgresql
  web:
    image: web_service:v1
    container_name: my_service
    ports:
      - 8080:8080
    depends_on:
      - postgres
volumes:
  pg-data: {}
```

### depends_on
>depends_on does not wait for db and redis to be “ready”
before starting web - only until they have been started.
If you need to wait for a service to be ready, see Controlling
startup order for more on this problem and strategies for solving it.

使用了depends_on在启动web这个容器是，并不会等待postgres容器进入ready状态，而只是等到它们被启动状态(running状态)了。 如果你需要等待直到postgres服务进入ready状态，需要额外的操作：wait-for 用来等待服务进入ready状态.


### 开始解决

#### 1.首先可以到上面提到的github上面下载wait-for.sh 脚本

```bash
#!/bin/sh

# The MIT License (MIT)
#
# Copyright (c) 2017 Eficode Oy
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

set -- "$@" -- "$TIMEOUT" "$QUIET" "$PROTOCOL" "$HOST" "$PORT" "$result"
TIMEOUT=15
QUIET=0
# The protocol to make the request with, either "tcp" or "http"
PROTOCOL="tcp"

echoerr() {
  if [ "$QUIET" -ne 1 ]; then printf "%s\n" "$*" 1>&2; fi
}

usage() {
  exitcode="$1"
  cat << USAGE >&2
Usage:
  $0 host:port|url [-t timeout] [-- command args]
  -q | --quiet                        Do not output any status messages
  -t TIMEOUT | --timeout=timeout      Timeout in seconds, zero for no timeout
  -- COMMAND ARGS                     Execute command with args after the test finishes
USAGE
  exit "$exitcode"
}

wait_for() {
  case "$PROTOCOL" in
    tcp)
      if ! command -v nc >/dev/null; then
        echoerr 'nc command is missing!'
        exit 1
      fi
      ;;
    wget)
      if ! command -v wget >/dev/null; then
        echoerr 'nc command is missing!'
        exit 1
      fi
      ;;
  esac

  while :; do
    case "$PROTOCOL" in
      tcp)
        nc -z "$HOST" "$PORT" > /dev/null 2>&1
        ;;
      http)
        wget --timeout=1 -q "$HOST" -O /dev/null > /dev/null 2>&1
        ;;
      *)
        echoerr "Unknown protocol '$PROTOCOL'"
        exit 1
        ;;
    esac

    result=$?

    if [ $result -eq 0 ] ; then
      if [ $# -gt 7 ] ; then
        for result in $(seq $(($# - 7))); do
          result=$1
          shift
          set -- "$@" "$result"
        done

        TIMEOUT=$2 QUIET=$3 PROTOCOL=$4 HOST=$5 PORT=$6 result=$7
        shift 7
        exec "$@"
      fi
      exit 0
    fi

    if [ "$TIMEOUT" -le 0 ]; then
      break
    fi
    TIMEOUT=$((TIMEOUT - 1))

    sleep 1
  done
  echo "Operation timed out" >&2
  exit 1
}

while :; do
  case "$1" in
    http://*|https://*)
    HOST="$1"
    PROTOCOL="http"
    shift 1
    ;;
    *:* )
    HOST=$(printf "%s\n" "$1"| cut -d : -f 1)
    PORT=$(printf "%s\n" "$1"| cut -d : -f 2)
    shift 1
    ;;
    -q | --quiet)
    QUIET=1
    shift 1
    ;;
    -q-*)
    QUIET=0
    echoerr "Unknown option: $1"
    usage 1
    ;;
    -q*)
    QUIET=1
    result=$1
    shift 1
    set -- -"${result#-q}" "$@"
    ;;
    -t | --timeout)
    TIMEOUT="$2"
    shift 2
    ;;
    -t*)
    TIMEOUT="${1#-t}"
    shift 1
    ;;
    --timeout=*)
    TIMEOUT="${1#*=}"
    shift 1
    ;;
    --)
    shift
    break
    ;;
    --help)
    usage 0
    ;;
    -*)
    QUIET=0
    echoerr "Unknown option: $1"
    usage 1
    ;;
    *)
    QUIET=0
    echoerr "Unknown argument: $1"
    usage 1
    ;;
  esac
done

if ! [ "$TIMEOUT" -ge 0 ] 2>/dev/null; then
  echoerr "Error: invalid timeout '$TIMEOUT'"
  usage 3
fi

case "$PROTOCOL" in
  tcp)
    if [ "$HOST" = "" ] || [ "$PORT" = "" ]; then
      echoerr "Error: you need to provide a host and port to test."
      usage 2
    fi
  ;;
  http)
    if [ "$HOST" = "" ]; then
      echoerr "Error: you need to provide a host to test."
      usage 2
    fi
  ;;
esac

wait_for "$@"
```

#### 2.在docker-compose.yml中引入脚本wait-for.sh

```yaml
version: '2'

services:
  postgres:
    image: postgres
    container_name: my_postgres
    restart: unless-stopped
    command:
      - 'postgres'
      - '-c'
      - 'max_connections=100'
      - '-c'
      - 'shared_buffers=256MB'
    environment:
      POSTGRES_DB: my_db1
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpassword
    ports:
      - 5432:5432
    volumes:
      - pg-data:/data/postgresql
  web:
    image: web_service:v1
    container_name: my_service
    ports:
      - 8080:8080
    depends_on:
      - postgres
    volumes:
      - ./wait-for.sh:/wait-for.sh
    command: ["./wait-for.sh", "postgres:5432","--", "./app" ,"--config.path=config.json"]
volumes:
  pg-data: {}
```

额外说明：
```cmd
command 中的  ./app --config.path = config.json 是web服务的启动命令

配置完启动docker-compose

docker-compose up

如果出现了：wait-for.sh 没有执行权限的问题可以先进行授权：
chmod 777 wait-for.sh

此时重新执行：
docker-compose up
```

成功！  
好好学习，天天向上！