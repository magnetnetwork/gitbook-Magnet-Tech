## 使用 ELK stack 分析 Ruby On Rails 日志
---

### ELK Stack

ELK Stack 是 Elasticsearch、Logstash、Kibana 三个开源软件的组合。在实时数据检索和分析场合，三者通常是配合共用，而且又都先后归于 Elastic.co 公司名下，故有此简称。

[ELK Stack官方指南](http://kibana.logstash.es/content/) 在最近两年迅速崛起，成为机器数据分析，或者说实时日志处理领域，开源界的第一选择.

####工作流程：

在需要收集日志的所有服务器上安装 Filebeat, (Filebeat 是基于原先 logstash-forwarder 的源码改造出来的。
换句话说：Filebeat 就是新版的 logstash-forwarder，也是 ELK Stack 在 shipper 端的第一选择。),
Filebeat 负责将日志发送给 Logstash Server 端(使用 lumberjack 网络协议), Logstash 接收到日志后，对日志进行分析，过滤
然后交给 ElasticSearch, 我们可以通过 Elasticsearch 对日志进行检索，同时 Kibana 会将日志可视化，生成友好的 Web 界面。


![elk stack](/images/elk_stack.png/)


####安装步骤：

总体分为3个部分:
1. 部署 logstash server 和 kibana server, 新开一个主机部署这两个服务
  * 安装 JAVA8 (Elasticsearch 和 Logstash 需要 Java，所以我们需要安装。)
```
    sudo add-apt-repository -y ppa:webupd8team/java (添加 Oracle JavaPPA 到 apt)
    sudo apt-get update
    sudo apt-get -y install oracle-java8-installer
```
  * 安装 kibana 和 logstash
```
    echo "deb http://packages.elastic.co/kibana/4.4/debian stable main" | sudo tee -a /etc/apt/sources.list.d/kibana-4.4.x.list
    echo 'deb http://packages.elastic.co/logstash/2.2/debian stable main' | sudo tee /etc/apt/sources.list.d/logstash-2.2.x.list
    sudo apt-get update
    sudo apt-get install kibana logstash
    sudo service kibana start   # 相关配置文件位于 /opt/kibana/config/kibana.yml
    sudo service logstash start # 相关配置文件位于 /etc/logstash/conf.d/logstash.conf
```
2. 部署 Elasticsearch 服务
  * 目前使用的是青云的 Elasticsearch 服务
3. 在 Web server 上安装并配置 Filebeat，使其能将日志发送给 Logstash Server
  * 安装 filebeat
```
    curl -L -O https://download.elastic.co/beats/filebeat/filebeat_1.3.1_amd64.deb
    sudo dpkg -i filebeat_1.3.1_amd64.deb
    vi /etc/filebeat/filebeat.yml
    sudo service filebeat start
```

####配置：

* logstash.conf
```
    input {
      beats {
        port => 5044
      }
    }

    output {
      elasticsearch {
        hosts => ["192.168.100.21:9200"]
      }
    }
```
* filebeat.conf
```
   filebeat:
     prospectors:
        paths:
          - /home/deploy/rails/magnet/shared/log/staging.log
        input_type: log
     logstash:
        hosts ["ip:port"]
```

#### 遇到的问题及解决

1.  一直不能通过 sudo service kibana start 启动 kibana

  `是因为我把 /opt/kibana 目录的 owner 设置成了 ubuntu, 其实它应该设置成 kibana:kibana`

2. Kibana flapping between red and green, Kibana Memory 80+%

   `vi /opt/kibana/bin/kibana`

   `将最后一行换成： exec "${NODE}" "${NODE_OPTIONS:=--max-old-space-size=200}" "${DIR}/src/cli" ${@}`
