# Logstash


Linux客户端日志收集

使用Filebeat进行日志收集
① 需安装filebeat
② 编辑/etc/filebeat/filebeat.yml文件,定义filename和svrip字段，将日志输出至一级logstash服务器10.88.60.110:5000
③ 重启filebeat服务

使用syslog、rsyslog进行日志收集

1.log类型

messages: 该日志文件是许多进程日志文件的汇总，从该文件可以看出任何入侵企图或成功的入侵。
secure: 该日志文件记录与安全相关的信息。
cron: 该日志文件记录crontab守护进程crond所派生的子进程的动作，前面加上用户、登录时间和PID，以及派生出的进程的动作。
yum:该日志文件记录使用yum安装的软件包信息
boot.log: 该日志文件记录了系统在引导过程中发生的事件，就是Linux系统开机自检过程显示的信息。
maillog: 该日志文件记录了每一个发送到系统或从系统发出的电子邮件的活动。
xferlog: 该日志文件记录FTP会话，可以显示出用户向FTP服务器或从服务器拷贝了什么文件。
spool


2.二进制文件类型


lastlog: 该日志文件记录最近成功登录的事件和最后一次不成功的登录事件，由login生成。
wtmp: 该日志文件永久记录每个用户登录、注销及系统的启动、停机的事件。
utmp: 该日志文件记录有关当前登录的每个用户的信息。

对于log类型的日志，直接进行收集
对于二进制文件类型，准备

/etc/filebeat/filebeat.yml
filebeat.prospectors:
- paths:
    - '/var/log/messages'
  fields_under_root: true
  fields:
    filename: 'messages'
    svrip: '10.88.142.203'

- paths:
    - '/var/log/boot.log'
  fields_under_root: true
  fields:
    filename: 'boot'
    svrip: '10.88.142.203'

- paths:
    - '/var/log/yum.log'
  fields_under_root: true
  fields:
    filename: 'yum'
    svrip: '10.88.142.203'

- paths:
    - '/var/log/cron'
  fields_under_root: true
  fields:
    filename: 'cron'
    svrip: '10.88.142.203'

- paths:
    - '/var/log/secure'
  fields_under_root: true
  fields:
    filename: 'secure'
    svrip: '10.88.142.203'

- paths:
    - '/var/log/maillog'
  fields_under_root: true
  fields:
    filename: 'maillog'
    svrip: '10.88.142.203'

- paths:
    - '/var/log/xferlog'
  fields_under_root: true
  fields:
    filename: 'xferlog'
    svrip: '10.88.142.203'

- paths:
    - '/var/log/spooler'
  fields_under_root: true
  fields:
    filename: 'spooler'
    svrip: '10.88.142.203'

output.logstash:
  hosts: ["10.88.60.110:5000"]

========公募/资管网========
output.logstash:
  hosts: ["172.20.60.110:5000"]
=======================





一级Logstash日志汇总

使用logstash进行日志汇总
① 需安装logstash、filebeat
② 编辑/etc/logstash/conf.d/下的input、output配置文件，将客户端日志输出到/opt/logstash/%{+YYYY-MM}/svr_filebeat/%{[svrip]}/%{[filename]}.log
③ 编辑/etc/filebeat/filebeat.yml配置文件，将日志进行汇总发送至二级Logstash服务器


/etc/logstash/conf.d/100-syslog-input.conf（无需修改）

syslog {
    type => "svr_aix"
    port => 5140
    }
}


/etc/logstash/conf.d/101-filebeats-input.conf（无需修改）
input {
  beats {
    port => 5000
  }
}

/etc/logstash/conf.d/302-svrfilebeats-output.conf
output {
        if [filename] in ["messages", "boot", "yum", "cron", "secure", "maillog", "xferlog", "spooler"] {
            file {
                path => "/opt/logstash/%{+YYYY-MM}/svr_filebeat/%{[svrip]}/%{[filename]}.log"
                codec => line { format => "@filename:%{filename} @svrip:%{svrip} @message:%{message} " }
                 }
        }
}
/etc/logstash/conf.d/303-svrsyslog-output.conf

output {
    if [type] in  ["svr_aix", "svr_linux"]  {
    file {
      path => "/opt/logstash/%{+YYYY-MM}/svr_syslog/%{type}/%{host}.log"
      codec => line { format => "%{@timestamp} severity_label:%{severity_label} message:%{message}"}
    }
  }
}


/etc/filebeat/filebeat.yml
#===========================Server Syslog==================================

- paths:
  - '/opt/logstash/*/svr_filebeat/*/*.log'
  fields_under_root: true
  fields:
    type: 'svr_filebeat'

- paths:
  - '/opt/logstash/*/svr_syslog/svr_aix/*.log'
  fields_under_root: true
  fields:
    type: 'svr_aix'

- paths:
  - '/opt/logstash/*/svr_syslog/svr_linux/*.log'
  fields_under_root: true
  fields:
    type: 'svr_linux'


#==========================================================================



二级Logstash日志汇总
① 需安装logstash
②编辑/etc/logstash/conf.d/下的input、filter、output配置文件，将客户端日志过滤输出到elasticsearch

/etc/logstash/conf.d/100-filebeat-input.conf（无需修改）
input {
  beats {
    port => 5044
  }
}

/etc/logstash/conf.d/300-es-output.conf（无需修改）
output {
    elasticsearch {
        hosts => ["10.88.60.112:9200"]
        index => "%{type}-%{+YYYY-MM}"
        document_type => "%{type}"
    }   
}

/etc/logstash/conf.d/202-svr_syslog-filter.conf
filter {
    if [type] == "svr_filebeat" {
       grok {
         match => [ "source", ".*%{IP:svrip}/%{DATA:filetype}.log$" ]
       }
    }
    if [type] == "svr_aix" {
       grok {
         match => {"source" => "%{IP:svrip}"}
       }
       grok {
         match => {"message" => "%{TIMESTAMP_ISO8601:logtime} %{DATA}:%{DATA:severity_label} %{DATA}:%{GREEDYDATA:log_message}"}
       }
       date {
         match => [ "logtime", "yyyy-MM-dd'T'HH:mm:ss.SSSZ" ]
       }
    }
    if [type] == "svr_linux" {
       grok {
         match => {"source" => "%{IP:svrip}"}
       }
       grok {
         match => {"message" => "%{TIMESTAMP_ISO8601:logtime} %{DATA}:%{DATA:severity_label} %{DATA}:%{GREEDYDATA:log_message}"}
       }
       date {
         match => [ "logtime", "yyyy-MM-dd'T'HH:mm:ss.SSSZ" ]
       }
    }
}



