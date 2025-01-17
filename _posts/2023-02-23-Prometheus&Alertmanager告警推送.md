[https://www.cnblogs.com/CKExp/p/17149389.html](https://www.cnblogs.com/CKExp/p/17149389.html)


## 前言

尽管可以通过可视化数据监控系统运行状态，但我们无法时刻关注系统运行，因此需要一些实时运行的工具能够辅助监控系统运行，当系统出现运行问题时，能够通知我们，以此确保系统稳定性，告警便是作为度量指标监控中及其重要的一环。

## Prometheus告警介绍

在Prometheus中，告警模块为Alertmanager，可以提供多种告警通道、方式来使得系统出现问题可以推送告警消息给相关人员。

![141209821_0eb57afa-0366-4a7a-90af-4df9f70bf6e2](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141209821_0eb57afa-0366-4a7a-90af-4df9f70bf6e2.png)
Prometheus Server中的告警规则会向Alertmanager发送。Alertmanager管理这些告警，包括进行去重，分组和路由，以及告警的静默和抑制，通过电子邮件、实时通知系统和聊天平台等方法发送通知。

* 定义告警规则(Alerting rule)

* 告警模块消息推送(Alertmanager)

* 告警推送(Prometheus server推送告警规则到Alertmanager)

## Alertmanager

Alertmanager是一个独立的告警模块，接收Prometheus server发来的告警规则，通过去重、分组、静默和抑制等处理，并将它们通过路由发送给正确的接收器。

### 核心概念

* 分组(Grouping): 是Alertmanager把同类型的告警进行分组，合并多条告警合并成一个通知。当系统中许多实例出问题时又或是网络延迟、延迟导致网络抖动等问题导致链接故障，会引发大量告警，在这种情况下使用分组机制，把这些被触发的告警合并为一个告警进行通知，从而避免瞬间突发性的接受大量告警通知，使得相关人员可以对问题快速定位，而不至于淹没在告警中。

* 抑制(Inhibition): 是当某条告警已经发送，停止重复发送由此告警引发的其他异常或故障的告警机制，以避免收到过多无用的告警通知。

* 静默(Silences): 可对告警进行静默处理的简单机制。对传进来的告警进行匹配检查，如果告警符合静默的配置，Alertmanager 则不会发送告警通知。

### 设置Alertmanager.yml

```plain
cd /opt
mkdir alertmanager
cd alertmanager
touch alertmanager.yml
```
配置文件主要包含以下几个部分：
* 全局配置（global）：用于定义一些全局的公共参数，如全局的SMTP配置，Slack配置等内容；

* 模板（templates）：用于定义告警通知时的模板，如HTML模板，邮件模板，企业微信模板等；

* 告警路由（route）：根据标签匹配，确定当前告警应该如何处理；

* 接收人（receivers）：接收人是一个抽象的概念，它可以是一个邮箱也可以是微信，Slack或者Webhook等，接收人一般配合告警路由使用；

配置文件内容：

```plain
global:
  resolve_timeout: 2m
  smtp_smarthost: 'smtp.qq.com:465'
  smtp_from: your@qq.com
  smtp_auth_username: your@qq.com
  smtp_auth_password: 授权码
templates:
  - /etc/alertmanager/template/*.tmpl
route:
  group_by:
    - alertname_wechat
  group_wait: 10s
  group_interval: 10s
  receiver: wechat
  repeat_interval: 1h
receivers:
  - name: wechat
    email_configs:
      - to: otheremail@outlook.com
        send_resolved: true
    wechat_configs:
      - corp_id: wechat_corp_id
        to_party: wechat_to_party
        agent_id: wechat_agent_id
        api_secret: wechat_apisecret
        send_resolved: true
```
此处设置推送消息到邮箱和企业微信，Alertmanager内置了对企业微信的支持。
[https://prometheus.io/docs/alerting/latest/configuration/#wechat_config](https://prometheus.io/docs/alerting/latest/configuration/#wechat_config)

### 设置消息模板

```plain
cd /opt/alertmanager
mkdir template
cd template
touch wechat.tmpl
```
编辑文件内容，设置模板格式
```plain
{{ define "wechat.default.message" }}
{{ range $i, $alert :=.Alerts }}
========监控报警==========
告警状态：{{   .Status }}
告警级别：{{ $alert.Labels.severity }}
告警类型：{{ $alert.Labels.alertname }}
告警应用：{{ $alert.Annotations.summary }}
告警主机：{{ $alert.Labels.instance }}
告警详情：{{ $alert.Annotations.description }}
触发阀值：{{ $alert.Annotations.value }}
告警时间：{{ $alert.StartsAt.Format "2023-02-19 10:00:00" }}
========end=============
{{ end }}
{{ end }}
```
### 部署Alertmanager

```plain
docker run -d -p 9093:9093 --name StarCityAlertmanager -v /opt/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml -v /opt/alertmanager/template:/etc/alertmanager/template docker.io/prom/alertmanager:latest
```
可访问[http://host:9093](http://localhost:9093/)
![141211008_2d58a357-728a-4b99-a818-bf370736000a](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141211008_2d58a357-728a-4b99-a818-bf370736000a.png)
### 告警通知测试

可通过请求Alertmanager Api模拟告警规则来推送告警通知。

```plain
curl --location 'http://Host:9093/api/v2/alerts' \
--header 'Content-Type: application/json' \
--data '[
    {
        "labels": {
            "severity": "Warning",
            "alertname": "内存使用过高",
            "instance": "实例1",
            "msgtype": "testing"
        },
        "annotations": {
            "summary": "node",
            "description": "请检查实例1",
            "value": "0.95"
        }
    },
    {
        "labels": {
            "severity": "Warning",
            "alertname": "CPU使用过高",
            "instance": "实例2",
            "msgtype": "testing"
        },
        "annotations": {
            "summary": "node",
            "description": "请检查实例2",
            "value": "0.90"
        }
    }
]'
```
发送完毕，可以在企业微信和邮件中收到告警通知，如在邮箱中收到信息。
![141212448_ea5097bd-027b-4d53-8ebf-f9f664f884fd](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141212448_ea5097bd-027b-4d53-8ebf-f9f664f884fd.png)
>注意：如果告警配置完毕，但测试时企业微信怎么也收不到消息，需要设置企业自建应用底部可信IP菜单
![141214080_77283594-b7f7-4ec3-81d3-fc99aecd1055](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141214080_77283594-b7f7-4ec3-81d3-fc99aecd1055.png)
## 告警规则

### 设置告警规则

```plain
cd /opt/prometheus
touch rules.yml
```
告警规则内容
```plain
groups:
  - name: example
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 10s
        labels:
          name: instance
          severity: Critical
        annotations:
          summary: ' {{ $labels.appname }}'
          description: ' The service stops running '
          value: '{{ $value }}%'
  - name: Host
    rules:
      - alert: HostMemory Usage
        expr: >-
          (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes +
          node_memory_Buffers_bytes + node_memory_Cached_bytes)) /
          node_memory_MemTotal_bytes * 100 >  80
        for: 10s
        labels:
          name: Memory
          severity: Warning
        annotations:
          summary: ' {{ $labels.appname }} '
          description: ' The instance memory usage exceeded 80%. '
          value: '{{ $value }}'
      - alert: HostCPU Usage
        expr: >-
          sum(avg without
          (cpu)(irate(node_cpu_seconds_total{mode!='idle'}[5m]))) by
          (instance,appname) > 0.65
        for: 10s
        labels:
          name: CPU
          severity: Warning
        annotations:
          summary: ' {{ $labels.appname }} '
          description: The CPU usage of the instance is exceeded 65%.
          value: '{{ $value }}'
      - alert: HostLoad
        expr: node_load5 > 4
        for: 10s
        labels:
          name: Load
          severity: Warning
        annotations:
          summary: '{{ $labels.appname }} '
          description: ' The instance load exceeds the default value for 5 minutes.'
          value: '{{ $value }}'
      - alert: HostFilesystem Usage
        expr: 1-(node_filesystem_free_bytes / node_filesystem_size_bytes) >  0.8
        for: 10s
        labels:
          name: Disk
          severity: Warning
        annotations:
          summary: ' {{ $labels.appname }} '
          description: ' The instance [ {{ $labels.mountpoint }} ] partitioning is used by more than 80%.'
          value: '{{ $value }}%'
      - alert: HostDiskio
        expr: 'irate(node_disk_writes_completed_total{job=~"Host"}[1m]) > 10'
        for: 10s
        labels:
          name: Diskio
          severity: Warning
        annotations:
          summary: ' {{ $labels.appname }} '
          description: ' The instance [{{ $labels.device }}] average write IO load of the disk is high in 1 minute.'
          value: '{{ $value }}iops'
      - alert: Network_receive
        expr: >-
          irate(node_network_receive_bytes_total{device!~"lo|bond[0-9]|cbr[0-9]|veth.*|virbr.*|ovs-system"}[5m])
          / 1048576  > 3
        for: 10s
        labels:
          name: Network_receive
          severity: Warning
        annotations:
          summary: ' {{ $labels.appname }} '
          description: ' The instance [{{ $labels.device }}] average traffic received by the NIC exceeds 3Mbps in 5 minutes.'
          value: '{{ $value }}3Mbps'
      - alert: Network_transmit
        expr: >-
          irate(node_network_transmit_bytes_total{device!~"lo|bond[0-9]|cbr[0-9]|veth.*|virbr.*|ovs-system"}[5m])
          / 1048576  > 3
        for: 10s
        labels:
          name: Network_transmit
          severity: Warning
        annotations:
          summary: ' {{ $labels.appname }} '
          description: ' The instance [{{ $labels.device }}] average traffic sent by the network card exceeds 3Mbps in 5 minutes.'
          value: '{{ $value }}3Mbps'
  - name: Container
    rules:
      - alert: ContainerCPU Usage
        expr: >-
          (sum by(name,instance)
          (rate(container_cpu_usage_seconds_total{image!=""}[5m]))*100) > 60
        for: 10s
        labels:
          name: CPU
          severity: Warning
        annotations:
          summary: '{{ $labels.name }} '
          description: ' Container CPU usage over 60%.'
          value: '{{ $value }}%'
      - alert: ContainerMem Usage
        expr: 'container_memory_usage_bytes{name=~".+"}  / 1048576 > 1024'
        for: 10s
        labels:
          name: Memory
          severity: Warning
        annotations:
          summary: '{{ $labels.name }} '
          description: ' Container memory usage exceeds 1GB.'
          value: '{{ $value }}G'
```
修改prometheus.yml，增加告警规则，如下alerting和rule_files部分，重启prometheus。
```plain
global:
  scrape_interval: 15s
  evaluation_interval: 15s


alerting:
  alertmanagers:
  - static_configs:
    - targets: ['Host:9093']
rule_files:
  - "rules.yml"  


scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - 'Prometheus Server Host:9090'
        labels:
          appname: Prometheus
  - job_name: node
    scrape_interval: 10s
    static_configs:
      - targets:
          - 'Metrics Host:9100'
        labels:
          appname: node
  - job_name: cadvisor
    static_configs:
      - targets:
          - 'Metrics Host:58080'
  - job_name: rabbitmq
    scrape_interval: 10s
    static_configs:
      - targets:
          - 'Metrics Host:9419'
        labels:
          appname: rabbitmq
```
再次访问prometheus可以看到告警规则
![141215289_9cec57f0-f3b0-494f-ae15-0b5be51eeb3f](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141215289_9cec57f0-f3b0-494f-ae15-0b5be51eeb3f.png)
### **告警状态**

|状态|说明|
|:----|:----|
|Inactive|待激活状态，度量指标处在合适范围内。|
|Pending|符合告警规则，但是低于配置的持续时间。这里的持续时间即rule里的FOR字段设置的时间。该状态下不发送告警通知。|
|Firing|符合告警规则，而且超出设置的持续时间。该状态下发送告警到Alertmanager。|

### 触发告警

当系统达到预定告警条件并超出设定的持续时间，则触发告警，推送告警消息到Alertmanager。

此处设置系统CPU使用率超过限定条件，可以在prometheus中看到CPU使用率告警规则达到Pending状态

![141216629_f74393b6-f7e6-4f7f-86f6-623f92129a67](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141216629_f74393b6-f7e6-4f7f-86f6-623f92129a67.png)
当超过设定的持续时间，状态变更到Firing，消息推送到Alertmanager

![141217969_a6bbf8e3-453b-4c39-ab2b-4499d793737d](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141217969_a6bbf8e3-453b-4c39-ab2b-4499d793737d.png)
在Alertmanager Web中可以看到推送过来的告警信息

![141219346_e6831ac1-6750-4bb6-82a6-76b9816aaaa0](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141219346_e6831ac1-6750-4bb6-82a6-76b9816aaaa0.png)
在企业微信和邮箱中也收到告警信息

![141221016_165794c8-f259-4071-b1bc-20de8bc24b67](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141221016_165794c8-f259-4071-b1bc-20de8bc24b67.png)
![141222812_3768ebb6-1f0a-4cd8-bf6b-396f9c3260da](https://raw.githubusercontent.com/SAssassin/document-img/main/img/20241127/141222812_3768ebb6-1f0a-4cd8-bf6b-396f9c3260da.png)
>2023-02-23,望技术有成后能回来看见自己的脚步

