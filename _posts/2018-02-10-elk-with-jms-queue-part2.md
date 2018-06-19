---
layout: post
title:  "ELK with JMS queue - Part II"
date:   2018-02-10 22:35:00 +0200
categories: elk logstash elastic jms activeMQ
---

## What's the point?

This is the second part of a quick guide to run a local pipeline for ingest messages from a JMS queue.
The [first part](https://ilittleangel.github.io/elk/logstash/elastic/jms/activemq/2018/02/01/elk-with-jms-queue-part1.html) is just for installations and check our enviroment.

> There is no need for an installation of Elasticsearch because we can use a pretty stdout for testing.

## JMS configuration

* We start to configure a JMS Server connection for our Weblogic servers, two in this case (`weblogic1`,`weblogic2`)
* We will put the configuration in a hocon configuration file named `jms.yml`.
* Notice that we have to get the WebLogic full client jar and include in the classpath `wlfullclient.jar` 

```hocon
weblogic1:
  :jndi_name: otewl10001.cosmos.es.ftgroup
  :jndi_context:
    java.naming.factory.initial: weblogic.jndi.WLInitialContextFactory
    java.naming.provider.url: t3://10.4.66.13:8011,10.8.66.117:8011
    java.naming.factory.url.pkgs: javax.naming:javax.jms
  :require_jars:
    - /tmp/wlfullclient.jar

weblogic2:
  :jndi_name: otewl10002.cosmos.es.ftgroup
  :jndi_context:
    java.naming.factory.initial: weblogic.jndi.WLInitialContextFactory
    java.naming.provider.url: t3://10.8.120.18:8011,10.8.120.119:8011
    java.naming.factory.url.pkgs: javax.naming:javax.jms
  :require_jars:
    - /tmp/wlfullclient.jar
```

## Installation of `logstash-filter-translate` pluging

* We are going to use this plugin to perform a join between a field of the jms input message and an external file.

```
logstash-plugin install logstash-filter-translate
```

## Edit the Logstash configuration file

* To setup a logstash pipeline we use a hocon configuration file `logstash-jms-pipeline.conf` in this case.
* The `input` section: 
  - we have two entries, one from each weblogic server.
  - and we have to copy the previous `jms.yml` file into logstash configuration path `/etc/logstash/conf.d/jms.yml`
* The `filter` section:
  - we apply a `grok` parser for a specific type of message
  - add and delete some fields
* The `output` section:
  - we define 3 different outputs (stdout, elasticsearch and kafka) but we can remove `elasticsearch` or `kafka` if we dont have a installation

```hocon
input {
     jms {
          type => "weblogic1"
          use_jms_timestamp => false
          yaml_file => "/etc/logstash/conf.d/jms.yml"
          yaml_section => "weblogic1"
          destination => "es1prdJMSModuleOUT!es1StreamAnalyticsOUT"
     }
     jms {
          type => "weblogic2"
          use_jms_timestamp => false
          yaml_file => "/etc/logstash/conf.d/jms.yml"
          yaml_section => "weblogic2"
          destination => "es2prdJMSModuleOUT!es2StreamAnalyticsOUT"
     }
}

filter {
     grok {
          match => [ "message", "%{NOTSPACE:http_method}.*/\?tfno=(?<tfno>.*)&texto=(?<texto>.*)
          &country=(?<country>[a-zA-Z]{3})&host=(?<host>[a-zA-Z0-9._-]+)&op=%{GREEDYDATA:op}
          &centro=%{GREEDYDATA:centro}&Numero=%{NOTSPACE:numero}&tipo=%{NOTSPACE:tipo}
          &user=%{USERNAME:user}(?:&err=%{GREEDYDATA:err}?|)&transId=(?<transid>[a-zA-Z0-9._]+)
          &TimeIn=%{NUMBER:timein:int}&RecepName=(?<recep_name>[a-zA-Z0-9._-]+)&Medio=%{GREEDYDATA:medio}
          &TansmisionType=%{WORD:transmision_type}&SeviceType=%{WORD:sevice_type}
          &ProtocolType=%{GREEDYDATA:protocol_type}&InOrOut=%{WORD:in_or_out}&DestinoType=%{WORD:destino_type}
          &OrigenType=%{WORD:origen_type}(?:&ModeloId=%{GREEDYDATA:modelo_id}?|)&origen=%{WORD:origen}
          &Ok=%{WORD:ok}&ev=%{GREEDYDATA:ev} HTTP" ]
     }

     if [ev] =~ "[a-zA-Z0-9]+#.*" {
          grok {
               match => [ "ev", "(?<id>[a-zA-Z0-9]+)#" ]
          }
     } else {
          grok {
               match => [ "ev", "(?<id>[a-zA-Z0-9]+)%" ]
          }
     }

     grok { match => [ "ev", "[^.*]{22}(?<command>[a-zA-Z0-9]{3})" ] }

     mutate { add_field => { "installation" => "%{id};%{country}" } }

     translate {
          field => "installation"
          destination => "location"
          dictionary_path => '/logstash-jms/location_installations.yaml'
     }

     if [location] {
          # add longitude field
          mutate { add_field => [ "lon", "%{location}" ] }
          mutate { split => ["lon", ";"] }
          mutate { replace => [ "lon", "%{[lon][0]}" ] }

          # add latitude field
          mutate { add_field => [ "lat", "%{location}" ] }
          mutate { split => ["lat", ";"] }
          mutate { replace => [ "lat", "%{[lat][1]}" ] }

          # remove location and installation fields
          #mutate { remove_field => [ "location" ] }
          #mutate { remove_field => [ "installation" ] }
     }

}

output {
    stdout { codec => rubydebug }
    elasticsearch {
          hosts => [ "localhost:9200" ]
          index => "logstash-jms-%{+YYYY.MM.dd}"
    }
    kafka {
          bootstrap_servers => "kafka01:9092,kafka02:9092,kafka03:9092"
          topic_id => "topicJms" 
    }
}
```


## Run Logstash and send JMS messages

We have saved the Logstash configuration file in `/etc/logstash/conf.d/` path but there is no need to save in this path, we can use wherever path.

For running from the command line
```
sudo /opt/logstash/bin/logstash -f /etc/logstash/conf.d/logstash-jms-pipeline.conf
```

After that we can send messages to JMS queue. In the [previous post](https://ilittleangel.github.io/elk/logstash/elastic/jms/activemq/2018/02/01/elk-with-jms-queue-part1.html) we have a step by step guide to send messages.
