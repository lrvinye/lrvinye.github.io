---
title: 使用Logstash同步Mysql的数据到ElasticSearch
date: 2020-06-23 23:48:51
updated: 2020-07-10 12:48:28
category: 中间件
tags: [k8s,Logstash,配置,搜索引擎,ES,ElasticSearch,MySql,数据库]
---

# 环境



>K8S 1.16.9
>
>logstash:7.6.2 镜像
>
>NFS 云硬盘



# 配置文件





## logstash 的基本配置格式

```conf
input { stdin { } } output { stdout {} }
```



> Logstash 官方提供的lostash关于apache,nginx应用的[日志处理样本](https://github.com/elastic/examples/tree/master/Common )



## 读取文件示例

```conf
input {
	file {
		#要读取的文件的位置
		path => "/Users/liuxg/data/cars.csv"
		
		#start_position指向beginning。如果对于一个实时的数据源来说，它通常是ending，这样表示它每次都是从最后拿到那个数据。
		start_position => "beginning"
		#sincedb_path通常指向一个文件。这个文件保存上次操作的位置。设置为/dev/null，表明不存储这个数据
		sincedb_path => "null"
	}
}
```





## 同步MySql 数据的配置文件示例

```conf
#conf.d/mysql.conf

input {
    jdbc {
      
      jdbc_driver_library => "/usr/share/logstash/jdbc.jar"
      jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
      # 8.0以上版本：一定要把serverTimezone=UTC天加上
      jdbc_connection_string => "jdbc:mysql://172.16.0.13:3306/test?characterEncoding=utf8&useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true"
      jdbc_user => "xxx"
      jdbc_password => "xxx"
      # 每5秒同步一次，6个*则为每秒同步一次  schedule =>秒 分 时 天 月 年  
      schedule => "*/5 * * * * *"
      # 使用updateTime作为界限，这样配置避免重复读
      statement => "SELECT * FROM user_info WHERE flag = 0 AND updateTime > UNIX_TIMESTAMP( :sql_last_value )*1000 AND updateTime < REPLACE( UNIX_TIMESTAMP( NOW(3) ),'.','');"

      #是否清除 last_run_metadata_path 的记录,如果为真那么每次都相当于从头开始查询所有的数据库记录
      clean_run => false

      #是否将 column 名称转小写
      lowercase_column_names => false
      
      # 是否需要记录某个column 的值,如果 record_last_run 为真,可以自定义我们需要 track 的 column 名称，此时该参数就要为 true. 否则默认 track 的是 timestamp 的值.
      #use_column_value => true
      #tracking_column_type => "bigint"
      # 如果 use_column_value 为真,需配置此参数. track 的数据库 column 名,该 column 必须是递增的.比如：ID.
      #tracking_column => "logId"

      #是否记录上次执行结果, 如果为真,将会把上次执行到的 tracking_column 字段的值记录下来,保存到 last_run_metadata_path 指定的文件中
      #record_last_run => true

      # #指定文件,来记录上次执行到的 tracking_column 字段的值
      # #比如上次数据库有 10000 条记录,查询完后该文件中就会有数字 10000 这样的记录,下次执行 SQL 查询可以从 10001 条处开始.
      # #我们只需要在 SQL 语句中 WHERE MY_ID > :sql_last_value 即可. 其中 :sql_last_value 取得就是该文件中的值(10000).
      #last_run_metadata_path => "/usr/share/logstash/run_metadata.d/mysql_lastrun"

      # #存放需要执行的 SQL 语句的文件位置
      # statement_filepath => "/etc/logstash/statement_file.d/my_info.sql"
  }

  stdin {
    }
}


filter {
    #jdbc默认json，暂时没找到修改方法
    json {
        source => "message"
        remove_field => ["message"]
    }
    mutate {  #需要移除的字段
        remove_field => "@timestamp"
        #remove_field => "type"
        remove_field => "@version"
 
    }
}

output {
   elasticsearch {
        # ES的IP地址及端口
        hosts => ["es-out:9200"]
        # 索引名称 可自定义
        index => "user-user_info"
        # 需要关联的数据库中有有一个id字段，对应类型中的id
        document_id => "%{userId}"
    }
  stdout {
        codec => json_lines
    }
}
```



# 启动 Logstash



- Docker方式



```sh
docker run -d -p 5044:5044 --name logstash -v conf:/usr/share/logstash/pipeline -v mysql-connector-java-8.0.15.jar:/usr/share/logstash/jdbc.jar logstash:7.6.2
```



- K8S方式

同上配置对应的目录挂载，端口开放