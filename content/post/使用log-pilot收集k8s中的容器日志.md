---
title: 使用 Log-pilot 收集 k8s 中的容器日志
slug: collect-container-logs-in-k8s-using-log-pilot
tags: [ELK, Docker, K8s]
date: 2022-12-11T10:41:51+08:00
---

容器时代越来越多的传统应用将会逐渐容器化，而日志又是应用的一个关键环节，那么在应用容器化过程中，如何方便快捷高效地来自动发现和采集应用的日志，如何与日志存储系统协同来高效存储和搜索应用日志。本文将主要跟大家分享下如何通过[Log-Pilot](https://github.com/AliyunContainerService/log-pilot)来采集容器的标准输出日志和容器内文件日志。<!--more-->

github地址： <https://github.com/AliyunContainerService/log-pilot>  
log-pilot官方介绍： <https://yq.aliyun.com/articles/674327>  
log-pilot官方搭建： <https://yq.aliyun.com/articles/674361?spm=a2c4e.11153940.0.0.21ae21c3mTKwWS>  
官方配置文件模版：<https://github.com/AliyunContainerService/log-pilot/tree/master/examples><!--more-->

在 k8s 中启动一个 deployment，配置文件 log-pilot.yaml 如下：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-pilot
  labels:
    k8s-app: log-pilot
spec:
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: log-pilot
  template:
    metadata:
      labels:
        app: log-pilot
    spec:
      tolerations:    # 这三行的作用是不会部署到 master 节点
      - key: node-role.kubernetes.io/master   # 这三行的作用是不会部署到 master 节点
        effect: NoSchedule    # 这三行的作用是不会部署到 master 节点
      containers:
      - name: log-pilot
        image: registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.7-filebeat  # 指定了最新的镜像
        env:
          - name: "LOGGING_OUTPUT"
            value: "kafka"
          - name: "KAFKA_BROKERS"
            value: "10.23.140.95:9092"  # 这里指定了传输到 kafka 的地址
          - name: "NODE_NAME"
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
        volumeMounts:
        - name: sock
          mountPath: /var/run/docker.sock
        - name: logs
          mountPath: /var/log/filebeat
        - name: state
          mountPath: /var/lib/filebeat
        - name: root
          mountPath: /host
          readOnly: true
        - name: localtime
          mountPath: /etc/localtime
        securityContext:
          capabilities:
            add:
            - SYS_ADMIN
      terminationGracePeriodSeconds: 30
      volumes:
      - name: sock
        hostPath:
          path: /var/run/docker.sock
      - name: logs
        hostPath:
          path: /var/log/filebeat
      - name: state
        hostPath:
          path: /var/lib/filebeat
      - name: root
        hostPath:
          path: /
      - name: localtime
        hostPath:
          path: /etc/localtime
```

启动，看他被调度到了哪个节点：

```bash
kubectl apply -f log-pilot.yaml
kubectl get pod -o wide | grep log-pilot
```

然后在启动一个测试用的 nginx，来收集 nginx 的日志做测试。配置文件 nginx-test.yaml 如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  selector:
    matchLabels:
      app: nginx-test
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      nodeName: 10.23.190.12
      containers:
      - name: nginx
        image: nginx:1.21-alpine
        imagePullPolicy: IfNotPresent
        env:
        - name: aliyun_logs_nginxtest  # 深坑，必须以 aliyun_logs 为开头
          value: "stdout"  # 这里指定收集容器 stdout 的输出日志
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
spec:
  selector:
    app: nginx-test
  ports:
  - port: 80
    targetPort: 80
    nodePort: 38888
  type: NodePort
```

查看被调度到了那个工作节点，然后去访问 nodePort 制造出几条日志：

```bash
kubectl get pod -o wide | grep nginx-test
kubectl get pod -o wide | grep log-pilot  # 查看一共多少 pod
kubectl logs log-pilot-4qj57
time="2022-12-08T19:32:54+08:00" level=debug msg="df6d99038672d8d12098247833bd0b2870f0942c8653bd6c5b7e6d83f823a3e0 has not log config, skip"
time="2022-12-08T19:32:54+08:00" level=debug msg="8ee1ce523604796d22e4be8e7c2788962f99b7c1f6eac92e479f9f10b5b03ee8 has not log config, skip"
time="2022-12-08T19:32:55+08:00" level=debug msg="a709830f96d7c8f9f01c74554a2fe1eb9141514b662774faf10ada520abb1e93 has not log config, skip"
time="2022-12-08T19:32:55+08:00" level=debug msg="44d5dce88d81e88de2ada3c32ced17e566a0d9a6ea8f0e7b4e4dd015d2c0a278 has not log config, skip"
time="2022-12-08T19:32:55+08:00" level=debug msg="3de0b1a3c3516af10cea7f25906cff26619fc667d7cf02224533c994f53fa247 has not log config, skip"
time="2022-12-08T19:32:55+08:00" level=debug msg="fcc6b8013ad759df9b8da5a3fb243779c5e5847c02f128c053d16ed6485a2295 has not log config, skip"
time="2022-12-08T19:32:55+08:00" level=debug msg="69f9c2044b534874dd03ed088bdd5a1fd95257cc956cf2d6c8c26672287dafb8 has not log config, skip"
time="2022-12-08T19:32:55+08:00" level=debug msg="6cf19d32da0ffe4dfb6bc1bcfc90cbe4010eca17fe75244f0cc10be154b00d6e has not log config, skip"
time="2022-12-08T19:32:55+08:00" level=debug msg="65fe73be91bfb6773c491d489c56fc0df61f7ba5314e422389394758261af0ac has not log config, skip"
time="2022-12-08T19:50:20+08:00" level=debug msg="Process container destroy event: 76fc6184a0dc8eeace500f17551c50d13a3aa75c76b90dcdb9fa5117c6caa7fc"
time="2022-12-08T19:50:20+08:00" level=info msg="begin to watch log config: 76fc6184a0dc8eeace500f17551c50d13a3aa75c76b90dcdb9fa5117c6caa7fc.yml"
time="2022-12-08T19:50:49+08:00" level=debug msg="Process container destroy event: 81b0c3603f980cbd4b346dfa6bd5c13ac40e723a71ed3bb4bc3930623e572a1b"
time="2022-12-08T19:50:49+08:00" level=info msg="begin to watch log config: 81b0c3603f980cbd4b346dfa6bd5c13ac40e723a71ed3bb4bc3930623e572a1b.yml"
time="2022-12-08T19:50:53+08:00" level=info msg="log config 76fc6184a0dc8eeace500f17551c50d13a3aa75c76b90dcdb9fa5117c6caa7fc.yml has been removed and ignore"
time="2022-12-08T19:50:53+08:00" level=info msg="log config 81b0c3603f980cbd4b346dfa6bd5c13ac40e723a71ed3bb4bc3930623e572a1b.yml has been removed and ignore"
time="2022-12-08T19:51:57+08:00" level=debug msg="Process container start event: 89cb27f0040b62c32756939aae91ef231d243fb18ba15a67309fa2da5f8aeaeb"
time="2022-12-08T19:51:57+08:00" level=debug msg="89cb27f0040b62c32756939aae91ef231d243fb18ba15a67309fa2da5f8aeaeb has not log config, skip"
time="2022-12-08T19:51:58+08:00" level=debug msg="Process container start event: 928fa2dbe79d7617cda619f51c556c91745d3202b8d590395424e920d276fa90"
time="2022-12-08T19:51:58+08:00" level=info msg="ignore checking the validity of kafka topic"
time="2022-12-08T19:51:58+08:00" level=info msg="logs: 928fa2dbe79d7617cda619f51c556c91745d3202b8d590395424e920d276fa90 = &{nginxtest /host/data/kube/docker/containers/928fa2dbe79d7617cda619f51c556c91745d3202b8d590395424e920d276fa90  nonex map[time_format:%Y-%m-%dT%H:%M:%S.%NZ] 928fa2dbe79d7617cda619f51c556c91745d3202b8d590395424e920d276fa90-json.log* map[index:nginxtest topic:nginxtest]  false true}"
time="2022-12-08T19:51:58+08:00" level=info msg="Reload filebeat"
time="2022-12-08T19:51:58+08:00" level=info msg="Start reloading"
time="2022-12-08T19:51:58+08:00" level=debug msg="do not need to reload filebeat"
```

这时候已经不显示 `has not log config, skip`，之前因为没写 `aliyun_log`，pod 识别不到要收集的 pod，所以一直跳过。  
由最后的输出看到，已经正常采集到了，通过 kafka-ui 看到，topic 也成功被创建，后面就是 logstash > es 的常规操作了。  
有的文章说 log-pilot 不支持 es7，但是，最终一套流程下来，我们的 es7 正常的收集到了日志，展示到了 kibana。  
[官方仓库下面有人在讨论这个事](https://github.com/AliyunContainerService/log-pilot/issues/235)，我猜测他们说的不支持，是不支持 log-pilot 直接输入到 es。