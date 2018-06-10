---
layout: post
title:  "ELK with JMS queue - Part I"
date:   2018-02-01 20:15:33 +0200
categories: elk logstash elastic jms activeMQ
---

## What's the point?

This publication is intended to simulate an environment of ingesting messages consumed from a JMS queue using Logstash and ActiveMQ.
> The guide assumes that we have a Logstash installation.


## Installations 

* Installing the Logstash plugin to read JMS queues

```bash
logstash-plugin install logstash-input-jms
```
![]({{"/assets/screenshot1.png"}})


* Installing the Logstash plugin to read JMS queues with stomp protocol

```bash
logstash-plugin install logstash-input-stomp
```
![]({{"/assets/screenshot2.png"}})

> **Note**: At the moment we do not know which of the two plugins we are going to use, so we installed the two plugins that allow us to read from a JMS queue


* Install Apache ActiveMQ

```bash
brew install apache-activemq
```
![]({{"/assets/screenshot3.png"}})

The ActiveMQ help

```bash
activemq help
```
![]({{"/assets/screenshot4.png"}})


## Configuration and boot

* We start the server or broker to send JMS messages

```bash
activemq start
```
Access to the administration web console

[http://localhost:8161/admin](http://localhost:8161/admin)

> **Note**: Link to the [ActiveMQ documentacion](http://activemq.apache.org/stomp.html) for the activation of Stomp (for localhost and this latest version of ActiveMQ no need to edit the xml file)


* We created an initial logstash configuration file with the stom input. Later we will add functionality to logstash to parse messages

```hocon
input {
     stomp {
        host => "localhost"
        destination => "JmsQueue"
    }
}
output {
     stdout { codec => rubydebug }
}
```


* Running Logstash

```bash
logstash -f logstash-jms-pipeline.conf --config.reload.automatic
```
![]({{"/assets/screenshot5.png"}})

Once we start Logstash a queue is automatically created in ActiveMQ and a stomp connection

![]({{"/assets/screenshot6.png"}})
![]({{"/assets/screenshot7.png"}})


* Send a message to that queue
We can use the ActiveMQ administration console

```
Mensaje IN:: GET /?tfno=0000026B124C5&texto=SDES051020401001126B124C50011606101158E1288612%23XTN005880NNNICE1EDK6ANUSMFFF00010000%2123A1&pais=ESP&host=radius06v&op=ORANGES&user=81349513&err=... HTTP/1.0>
```
![]({{"/assets/screenshot8.png"}})

As we click on the Send button we get the output in Logstash

![]({{"/assets/screenshot9.png"}})


* Finally, stop the broker

```bash
activemq stop
```
![]({{"/assets/screenshot10.png"}})



## Next steps 

To implement a real case, we will have to connect to your JMS server. Therefore, we will have to use the JMS pluging of Logstash.
For this you have to configure a `jms.yml` file.
![]({{"/assets/screenshot11.png"}})

