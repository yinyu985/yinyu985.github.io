---
title: Docker Compose 运行 Elastic Stack 8.X
slug: Docker-Compose-runs-Elastic-Stack-8.X
tags: [ELK, Docker]
date: '2023-09-28T12:20:25+08:00'
---

## 材料准备

如题，使用 Docker Compose 运行 Elastic Stack 8.X，五个月前还在前司做 7.9 版本的 ELK 迁移，如今已经是前司了，当时总共不到 20TB，最近问前同事数据量已经到 40TB 了，而且根据她发我的截图来看，三千多个索引，副本分片却只有两千多，不操心了，事不关己高高挂起，反正现在负责 ELK 的是新来的陌生人。今天看已经发到 8.10 版本了，来装一个感受下新特性。

[官方 Docker Compose](https://github.com/elastic/elasticsearch/blob/8.10/docs/reference/setup/install/docker/docker-compose.yml)

[官方安装指引文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#_start_a_single_node_cluster)

[额外操作参考博客](https://gigi.nullneuron.net/gigilabs/filebeat-elasticsearch-and-kibana-with-docker-compose/)

```yaml
version: "2.2"

services:
  setup:
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
    user: "0"
    command: >
      bash -c '
        if [ x${ELASTIC_PASSWORD} == x ]; then
          echo "Set the ELASTIC_PASSWORD environment variable in the .env file";
          exit 1;
        elif [ x${KIBANA_PASSWORD} == x ]; then
          echo "Set the KIBANA_PASSWORD environment variable in the .env file";
          exit 1;
        fi;
        if [ ! -f config/certs/ca.zip ]; then
          echo "Creating CA";
          bin/elasticsearch-certutil ca --silent --pem -out config/certs/ca.zip;
          unzip config/certs/ca.zip -d config/certs;
        fi;
        if [ ! -f config/certs/certs.zip ]; then
          echo "Creating certs";
          echo -ne \
          "instances:\n"\
          "  - name: es01\n"\
          "    dns:\n"\
          "      - es01\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          "  - name: es02\n"\
          "    dns:\n"\
          "      - es02\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          "  - name: es03\n"\
          "    dns:\n"\
          "      - es03\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          > config/certs/instances.yml;
          bin/elasticsearch-certutil cert --silent --pem -out config/certs/certs.zip --in config/certs/instances.yml --ca-cert config/certs/ca/ca.crt --ca-key config/certs/ca/ca.key;
          unzip config/certs/certs.zip -d config/certs;
        fi;
        echo "Setting file permissions";
        chown -R root:root config/certs;
        find . -type d -exec chmod 750 \{\} \;;
        find . -type f -exec chmod 640 \{\} \;;
        echo "Waiting for Elasticsearch availability";
        until curl -s --cacert config/certs/ca/ca.crt https://es01:9200 | grep -q "missing authentication credentials"; do sleep 30; done;
        echo "Setting kibana_system password";
        until curl -s -X POST --cacert config/certs/ca/ca.crt -u "elastic:${ELASTIC_PASSWORD}" -H "Content-Type: application/json" https://es01:9200/_security/user/kibana_system/_password -d "{\"password\":\"${KIBANA_PASSWORD}\"}" | grep -q "^{}"; do sleep 10; done;
        echo "All done!";
      '
    healthcheck:
      test: ["CMD-SHELL", "[ -f config/certs/es01/es01.crt ]"]
      interval: 1s
      timeout: 5s
      retries: 120

  es01:
    depends_on:
      setup:
        condition: service_healthy
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - esdata01:/usr/share/elasticsearch/data
    ports:
      - ${ES_PORT}:9200
    environment:
      - node.name=es01
      - cluster.name=${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es02,es03
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es01/es01.key
      - xpack.security.http.ssl.certificate=certs/es01/es01.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es01/es01.key
      - xpack.security.transport.ssl.certificate=certs/es01/es01.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.license.self_generated.type=${LICENSE}
    mem_limit: ${MEM_LIMIT}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'"
        ]
      interval: 10s
      timeout: 10s
      retries: 120

  es02:
    depends_on:
      - es01
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - esdata02:/usr/share/elasticsearch/data
    environment:
      - node.name=es02
      - cluster.name=${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es01,es03
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es02/es02.key
      - xpack.security.http.ssl.certificate=certs/es02/es02.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es02/es02.key
      - xpack.security.transport.ssl.certificate=certs/es02/es02.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.license.self_generated.type=${LICENSE}
    mem_limit: ${MEM_LIMIT}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'"
        ]
      interval: 10s
      timeout: 10s
      retries: 120

  es03:
    depends_on:
      - es02
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - esdata03:/usr/share/elasticsearch/data
    environment:
      - node.name=es03
      - cluster.name=${CLUSTER_NAME}
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es01,es02
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es03/es03.key
      - xpack.security.http.ssl.certificate=certs/es03/es03.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es03/es03.key
      - xpack.security.transport.ssl.certificate=certs/es03/es03.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.license.self_generated.type=${LICENSE}
    mem_limit: ${MEM_LIMIT}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'"
        ]
      interval: 10s
      timeout: 10s
      retries: 120

  kibana:
    depends_on:
      es01:
        condition: service_healthy
      es02:
        condition: service_healthy
      es03:
        condition: service_healthy
    image: docker.elastic.co/kibana/kibana:${STACK_VERSION}
    volumes:
      - certs:/usr/share/kibana/config/certs
      - kibanadata:/usr/share/kibana/data
    ports:
      - ${KIBANA_PORT}:5601
    environment:
      - SERVERNAME=kibana
      - ELASTICSEARCH_HOSTS=https://es01:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
      - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=config/certs/ca/ca.crt
    mem_limit: ${MEM_LIMIT}
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s -I http://localhost:5601 | grep -q 'HTTP/1.1 302 Found'"
        ]
      interval: 10s
      timeout: 10s
      retries: 120

volumes:
  certs:
    driver: local
  esdata01:
    driver: local
  esdata02:
    driver: local
  esdata03:
    driver: local
  kibanadata:
    driver: local
```

## 步骤分析

来欣赏下官方文档里面都干了些啥吧

- 创建了一个卷用来存储各种证书
- 判断 Elasticsearch 的密码和 Kibana 的密码是否设置（在 `.env` 文件中设置环境变量的值，包括密码，端口）
- 自签了一个 CA 然后给每个节点签发了证书，不了解这个流程的，自行谷歌

在 YAML 中运行 bash，使用 bash 执行命令为每个节点生成配置文件，又是 YAML 格式的，俄罗斯套娃。

关于 bash 的处理，我注意到一点

```bash
        if [ x${ELASTIC_PASSWORD} == x ]; then
          echo "Set the ELASTIC_PASSWORD environment variable in the .env file";
          exit 1;
        elif [ x${KIBANA_PASSWORD} == x ]; then
          echo "Set the KIBANA_PASSWORD environment variable in the .env file";
          exit 1;
        fi;
        if [ ! -f config/certs/ca.zip ]; then
          echo "Creating CA";
          bin/elasticsearch-certutil ca --silent --pem -out config/certs/ca.zip;
          unzip config/certs/ca.zip -d config/certs;
        fi;
```

前面使用了一种不常见（我不常见）的方式来判断环境变量是否被定义，后面使用了常规的 `-f` 来判断压缩文件是否存在，[我查了](https://linuxhint.com/bash_operator_examples/#o48)下明明有 `-z` 和 `-n` 这两种操作符可以用来判断，问了下 GPT

> ```
> if [ x${ELASTIC_PASSWORD} == x ]; then
>   echo "Set the ELASTIC_PASSWORD environment variable in the .env file";
>   exit 1;
> elif [ x${KIBANA_PASSWORD} == x ]; then
>   echo "Set the KIBANA_PASSWORD environment variable in the .env file";
>   exit 1;
> fi;
> ```
>
> ————————————
>
> 在这段 shell 脚本中为什么要这样判断环境变量有没有被设置呢？不是有更简单的 `-z` 和 `-n` 吗？这么判断是处于什么考虑的呢？我相信脚本的作者有自己的想法的

> 你提到的这段 shell 脚本是在判断 `ELASTIC_PASSWORD` 和 `KIBANA_PASSWORD` 这两个环境变量是否被设置。
>
> 之所以要用 `[ x${ELASTIC_PASSWORD} == x ]` 而不直接用 `-z` 来判断，主要出于以下考虑：
>
> 1. 处理空变量值的情况
>
> 如果环境变量设置为空值，像这样：
>
> ```
> ELASTIC_PASSWORD=""
> ```
>
> 使用 `-z` 判断会误判为未设置，因为它判断的是字符串长度是否为 0。
>
> 而 `[ x${ELASTIC_PASSWORD} == x ]` 可以正确判断出为空值的情况。
>
> 2. 避免环境变量未设置的情况
>
> 如果环境变量本身未设置，直接用 `$ELASTIC_PASSWORD` 会展开为 null。
>
> 而 `[ x${ELASTIC_PASSWORD} == x ]` 可以避免这个问题，如果未设置就展开为 `x` 而不是 null。
>
> 3. 考虑到可读性
>
> 相比 `-z` 和 `-n`，像 `[ x${ELASTIC_PASSWORD} == x ]` 这样的写法更容易一眼看明白其目的——判断变量是否为空。
>
> 所以综合考虑处理空值、避免未设置的情况、可读性等因素，作者选择了这种看似比较复杂但更加严谨和可读的写法。
>
> 这在 shell 编程中是一种很常见和推荐的判断环境变量设置的最佳实践写法。

## 回归正题

正常情况下，设置完密码，不需要做任何修改 `docker compose -f "elasticsearch/docker-compose.yml" up -d` 就可以启动了，但是为了跑通整个流程，我们再加上一个 `filebeat` 的采集端，[参考博客](https://gigi.nullneuron.net/gigilabs/filebeat-elasticsearch-and-kibana-with-docker-compose/)

```yaml
filebeat:
  depends_on:
    es01:
      condition: service_healthy
    es02:
      condition: service_healthy
    es03:
      condition: service_healthy
  image: docker.elastic.co/beats/filebeat:${STACK_VERSION}
  container_name: filebeat
  volumes:
    - ./filebeat.yml:/usr/share/filebeat/filebeat.yml  # 当前目录新建 filebeat 配置文件
    - certs:/usr/share/elasticsearch/config/certs
  environment:
    - ELASTICSEARCH_HOSTS=https://es01:9200
    - ELASTICSEARCH_USERNAME=elastic
    - ELASTICSEARCH_PASSWORD=${ELASTIC_PASSWORD}
    - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=config/certs/ca/ca.crt
```

然后集群内不通信也得给 `filebeat` 生成一份证书，修改生成证书的部分如下

```yaml
"  - name: es03\n"\
"    dns:\n"\
"      - es03\n"\
"      - localhost\n"\
"    ip:\n"\
"      - 127.0.0.1\n"\
"  - name: filebeat\n"\
"    dns:\n"\
"      - filebeat\n"\
"      - localhost\n"\
"    ip:\n"\
"      - 127.0.0.1\n"\
> config/certs/instances.yml;
```

然后开始配置 `filebeat`

```yaml
filebeat.inputs:
  - type: filestream
    id: my-application-logs
    enabled: true
    paths:
      - /var/log/*.log  # filebeat 的采集路径

output.elasticsearch:
  hosts: '${ELASTICSEARCH_HOSTS:elasticsearch:9200}'
  username: '${ELASTICSEARCH_USERNAME:}'
  password: '${ELASTICSEARCH_PASSWORD:}'
  ssl:
    certificate_authorities: "/usr/share/elasticsearch/config/certs/ca/ca.crt"
    certificate: "/usr/share/elasticsearch/config/certs/filebeat/filebeat.crt"
    key: "/usr/share/elasticsearch/config/certs/filebeat/filebeat.key"
```

好了，正常启动，大功告成，这个官方的 Docker Compose 中还用了 `healthcheck` 和 `mem_limit` 还有 `ulimit`，健康检查自然不用多说，其他的两个，一个是限制容器可以使用的最大内存量，另一个如下。

> `ulimits`（用户限制）：`ulimits` 是一个用于设置容器内进程的用户限制的配置项。这些限制通常用于控制容器内进程可以使用的系统资源，如内存、文件句柄、CPU 等。在您的示例中，`ulimits` 设置了 `memlock` 的软限制和硬限制都为 `-1`。软限制和硬限制之间的区别如下：
>
> - 软限制（soft limit）：这是一个进程可以使用的资源的最大值，但它可以超过这个值，只要系统资源允许。软限制可以被进程自身修改。
> - 硬限制（hard limit）：这是资源的绝对最大值，超过这个值的请求将被拒绝。硬限制只能由超级用户或管理员修改。
>
> 在这种情况下，`memlock` 的软限制和硬限制都设置为 `-1`，表示容器内的进程可以锁定无限量的内存，通常用于需要大量内存的应用程序。

## 完