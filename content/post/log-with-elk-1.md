---
title: "Working with application logs"
featured_image: "/images/mystreet.jpeg"
date: 2020-07-04T20:49:32-05:00
tags: ["blog", "elk", "log"]
toc: false
draft: true
---

A small tutorial to work with application logs and analyze them with Elastic and Kibana. In a microservices application architecture, it is important to analyze the logs to understand the health of the application. We can extract several stats from the logs when a log is a strucApplication logs can be generated in multiple ways. They can be either simple logs or print to console (with no time stamps).

## Collecting logs
- **Log files** - Most applications default behaviour is to write the logs on the file system. They may rolled up by date or size. A simple [file beat](https://www.elastic.co/beats/filebeat) process in the server can export the logs to either [logstash](https://www.elastic.co/logstash) or [fluentd](https://www.fluentd.org/)  or directly to [elastic](https://www.elastic.co/elasticsearch/).
	- Advantages: 
		- Simple file system
		- Backups available as long as the file is purged.
		- No code changes required.
	- Disadvantages:
		- As many as file exporter processes required to export the logs.
		- May miss console outputs.
		- In container environment, we need to create a volume for logs and mount them when spinning a container.
- **Message brokers** - When writing logs into file system, there is a possibility that we need to clean up the disk space regularly. It can be cumbersome in containers (creating a volume and mount to containers). There are language specific log appenders available to push the logs directly from application. 
	- Advantages: 
		- No file system
		- Log appenders can asynchronysly push logs to message brokers
		- Messages can be dropped after TTL 
		- If the upstream processes are not accessible, logs won't be lost
	- Disadvantages
		- Need infrastructure for message brokers.
- **Log drivers** - This is more suitable option for legacy application where there are limitted accessability to code or change in application is forbidden. 
	- Advantages: 
		- No code change required.
		- Everything that application emits including console prints.
		- Several log drivers comes by default with docker runtime.
		- No additional infrastructure.
	- Disadvantages:
		- There may be data lose with some protocols.
		- Some log drivers can cause application hung when the upstream is not processing the data correctly or if not designed correctly.

> Collecting logs via message brokers is most reliable option when considering failures in each layer even we need infrastructure for brokers.

## Best practices
- Use JSON: Json format logs can be helpful rather than unstructured data. 
- One log per function: Try to write one log function per exit point of the function. 
- Check for isDebugEnabled: String concatenation is super expensive in any language. Do you message construction after checking the log level as most of the time the debug would be disabled.
- In microservices environment, keep the kafka topic name with some prefix. Just one process is sufficient in processing multiple topics.
- Keep index templates and enable index policies in Kibana. If logs needs to be preserved for long period of time, consider moving them to cold storage after certain period. We can still search them but stored in inexpensive storage.

## Sample logstash config
Sample log stash configuration consuming messages from multiple topics of kafka and ingest into individual indices in elastic search.

    input {
        kafka {
            codec => "json"
            bootstrap_servers => "broker1:9092,broker2:9092"
            topics_pattern => "prefix-.*-logs"
            group_id => "consumer_group_id"
            auto_offset_reset => "earliest"
            decorate_events => true
            metadata_max_age_ms => 60000
            heartbeat_interval_ms => "250000"
            session_timeout_ms => "300000"
            poll_timeout_ms => "30000"
            consumer_threads => 6
        }
    }

    filter {
     kv {
         #default source field is message
         #source => "message"
         field_split_pattern => "([,\s]+)(?=\w+\s*\=)"
         transform_key => "lowercase"
         recursive => "true"
         trim_value => "<>\[\],"

    }
    grok {
        match => [ "message", "%{GREEDYDATA}" ]
    }

    mutate {
        #some languages emits json logs with double quotes. we need to change them to
        #single quote before parsing as json.
        gsub => [
                "msg", '"{', "'{",
                "msg", '}"', "}'"
            ]
    }
    json {
        source => "msg"
    }
    date {
        match => ["[@metadata][kafka][timestamp]", "UNIX_MS"]
        target =>  "@timestamp"
       }
    ruby {
        code => "event.set('[@metadata][kafka][lc_topic]', event.get('[@metadata][kafka][topic]').split(/(?=[A-Z])/).map{|x| x.downcase }.join('_') )"
      }
    }

    output {
        elasticsearch{
            index => "%{[@metadata][kafka][lc_topic]}-%{+YYYY.MM.dd}"
            hosts => ["http://host1:9200", "http://host2:9200"]
        }
        #stdout {
            #codec => rubydebug { metadata => true }
        #}
    }
