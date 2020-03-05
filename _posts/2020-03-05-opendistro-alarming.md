---
layout: post
title:  "Open Distro for Elasticsearch (Alerting Module)"
date:   2020-03-05 20:00:00 +0200
categories: elastic opendistro
---

* [Terminos clave](#terminos-clave)
* [Jerarquia de objetos en OpenDistro Alarming](#jerarquia-de-objetos-en-opendistro-alarming)
* [Crear DESTINATIONS](#crear-destinations-slack)
* [Actualizar DESTINATIONS](#actualizar-destinations)
* [Las queries en Elasticsearch](#las-queries-en-elasticsearch)
* [Crear MONITORS](#crear-monitors)
* [Actualizar MONITORS](#actualizar-monitors)
* [Ejecutar un MONITOR](#ejecutar-un-monitor)
* [Acknowledge alert (dar por enterado)](#acknowledge-alert-dar-por-enterado)


## Terminos clave

| Termino     | Definicion |
|:------------|:-----------|
| Monitor     | **JOB** que corre dentro de Opendistro bajo una planificacion temporal y ejecuta queries a Elaticsearch. El resultado de las queries es el input de uno o varios **TRIGGER**s. |
| Trigger     | **CONDICION** que si coincide con el resultado de un **MONITOR**, desencadena una **ACTION**. |
| Action      | **INFORMACION** que manda un **MONITOR** cuando salta el **TRIGGER**. Tiene un **DESTINATION**, un mensaje y un body. |
| Destination | **DESTINO** reusable de una **ACTION**. Puede ser un **webhook URL**, Slack o Amazon Chime. |


## Jerarquia de objetos en OpenDistro Alarming

```
MONITOR
    ├── Inputs[]
    │    └── Search
    │
    └── Triggers[]
         ├── Condition
         └── Actions[]
                └── Destination
```

>Los objetos con [] significa que son un array, por tanto se pueden definir uno o varios.



## Crear DESTINATIONS (Slack)

* Lo primero es crear el DESTINATION ya que lo usaremos para definir las ACTIONS dentro de un TRIGGER.
* Usaremos el API Rest de Opendistro aunque tambien se puede crear desde el modulo de Alarming en la web de Kibana.
* Crearemos previamente un [Incoming Webhook de Slack](https://api.slack.com/messaging/webhooks) y usaremos la url generada.

#### HTTP Request
```sh
curl -k \
     -H "Authorization: Basic $(echo -n admin:admin | base64)" \
     -H "Content-Type: application/json" \
     -XPOST "https://es-opendistro-node1:9200/_opendistro/_alerting/destinations?pretty" \
     -d @destinations/slack_test.json
```

**_destinations/slack_test.json_**
```json
{
  "name": "slack-test",
  "type": "slack",
  "slack": {
   "url": "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX"
  }
}
```

#### HTTP Response
```json
{
  "_id" : "xx2mRnFGfH_hrbnTNI8c",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "destination" : {
    "type" : "slack",
    "name" : "slack-test",
    "schema_version" : 1,
    "last_update_time" : 1582198838115,
    "slack" : {
      "url" : "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX"
    }
  }
}
```
>De la respuesta, mas tarde usaremos el campo `_id` para crear el TRIGGER


## Actualizar DESTINATIONS

```sh
curl -k \
     -H "Authorization: Basic $(echo -n admin:admin | base64)" \
     -H "Content-Type: application/json" \
     -XPUT "https://es-opendistro-node1:9200/_opendistro/_alerting/destinations/<destination-id>?pretty" \
     -d @destinations/slack_test.json
```

**_destinations/slack_test.json_**
```json
{
  "name": "updated-destination",
  "type": "slack",
  "slack": {
   "url": "updated-slack-webhook"
  }
}
```



## Las queries en Elasticsearch

En Elasticsearch podemos hacer dos tipos de búsquedas:
* **_`QUERY`_**: 
    - Son búsquedas que nos devuelve un score.
    - Son para buscar tokens en un texto o frase, lo que se suele llamar full text search.
    - No lo hacemos sobre los _Keywords_, sino sobre los campos de tipo _Text_.
* **_`FILTER`_**: 
    - Son búsquedas sin puntuación. 
    - Se usan para busquedas binarias en las que o encuentras lo que buscas o no.
    - Busquedas mas rapidas y cacheadas.
    - Cuando busquemos sobre campos de tipo String, usaremos los Keywords, no los Text.
    - Lo que en el 99% de los casos usaremos para el alarmado.

> En Elasticsearch los campos de tipo String se dividen en Text y Keywords. Los primeros se analizan y se usan 
las busquedas de tipo QUERY y en los segundos no se analizan y se usan para busqudas FILTER.
  
En las búsquedas de Elasticsearch podemos usar varias condiciones de búsqueda:
- Si ponemos `must` se tiene que cumplir todas las condiciones
- si ponemos `should` basta con que se cumpla alguna para devolver resultados
  
Sabiendo este poquito sobre las búsquedas en Elasticsearch ya podemos crear las primeras queries para poder alarmar.



## Crear MONITORS

* La creación de monitores la podemos hacer desde la interfaz de Kibana y desde el API Rest de OpenDistro.
* En esta pequeña guía nos centraremos en el API pero tambien dejaremos disponibles queries para usarlas como templates si usamos la interfaz web de Kibana.

Cuando creamos un MONITOR, tanto si es con la API HTTP o con Kibana, tenemos que tener claro lo siguiente:
* La **query** que se va a ejecutar contra Elasticsearch.
* El o los **TRIGGER**s, con su condición, que será la que dispare una ACTION cuando esta devuelva True.
* La **ACTION** con el mensaje y el body que se mandará al DESTINATION.
* Si usamos el API Rest tenemos que construir un json body con todo (query, triggers, actions)

>La query de ejemplo está identada de forma mas exagerada para que se vea mejor. Es una query de ejemplo que calcula la media
en las ultimas 24h de un campo numérico `duration` aplicando un filtro por el termino `process_name.keyword`.

##### HTTP Request
```sh
curl -k \
     -H "Authorization: Basic $(echo -n admin:admin | base64)" \
     -H "Content-Type: application/json" \
     -XPOST "https://es-opendistro-node1:9200/_opendistro/_alerting/monitors?pretty" \
     -d @monitors/process1.json
```

**_monitors/process1.json_**
```json
{
  "type": "monitor",
  "name": "process1",
  "enabled": true,
  "schedule": {
    "period": {
      "interval": 1,
      "unit": "MINUTES"
    }
  },
  "inputs": [{
    "search": {
      "indices": ["my-index-*"],
      "query": {
                 "size": 0,
                 "query": {
                   "bool": {
                     "filter": [
                       {
                         "bool": {
                           "must": [
                             {
                               "range": {
                                 "ts": {
                                   "from": "now-1d/d",
                                   "to": "now/d"
                                 }
                               }
                             },
                             {
                               "term": {
                                 "process_name.keyword": {
                                   "value":"process1"
                                 }
                               }
                             }
                           ]
                         }
                       }
                     ]
                   }
                 },
                 "aggregations":{
                   "avg_events_received":{
                     "avg":{
                       "field":"duration"
                     }
                   }
                 }
               }
    }
  }],
  "triggers": [{
    "name": "test-trigger",
    "severity": "5",
    "condition": {
      "script": {
        "source": "ctx.results[0].hits.total.value > 1000",
        "lang": "painless"
      }
    },
    "actions": [{
      "name": "test-action",
      "destination_id": "xx2mRnFGfH_hrbnTNI8c",
      "message_template": {
         "source": "{ \"text\": \"Monitor {{ctx.monitor.name}} just entered alert status. Please investigate the issue. - Trigger: {{ctx.trigger.name}} - Severity: {{ctx.trigger.severity}} - Period start: {{ctx.periodStart}} - Period end: {{ctx.periodEnd}}\" }"
      },
      "throttle_enabled": false,
      "subject_template": {
        "source": "ALARM - process1 - duration > 1000"
      }
    }]
  }]
}
```

##### HTTP Response
```json
{
  "_id" : "fT5FZHBBfH_hfqpMQW_x",
  ...
    "name" : "process1",
    ...
}
```

>De la respuesta, nos quedamos el `_id` del monitor, para poder luego interactuar con el (activar, deactivar, acknowledge ..)
Aunque por supuesto tambien tenemos metodos GET para recuperar esta informacion. 



## Actualizar MONITORS

* Igual que la creacion de MONITORS pero usando el verbo `PUT` sobre un `monitor_id`.

```sh
curl -k \
    -H "Authorization: Basic $(echo -n admin:admin | base64)" \
    -H "Content-Type: application/json" \
    -XPUT "https://es-opendistro-node1:9200/_opendistro/_alerting/monitors/<monitor_id>?pretty" \
    -d @monitors/updated_monitor.json
```

## Ejecutar un MONITOR

* Tenemos para esto el endpoint `_execute`.

```sh
curl -k \
    -H "Authorization: Basic $(echo -n admin:admin | base64)" \
    -XPOST "https://es-opendistro-node1:9200/_opendistro/_alerting/monitors/<monitor_id>/_execute?pretty"
```


## Acknowledge alert (dar por enterado)

* Tenemos para esto el endpoint `_acknowledge`
* Para marcar una ALERT necesitamos el ID.
* Este lo recuperamos del indice de Elasticsearch `.opendistro-alerts`.
* Las alertas pueden estar ya en los estados `ERROR`, `COMPLETED` ó `ACKNOWLEDGED`. Si es el caso, la llamada al endpoint `_acknowledge` devolvera la alerta en un campo `failed` de tipo array.
* Se puede marcar con `ACKNOWLEDGED` una o varias alertas a la vez.

```sh
curl -k \
    -H "Authorization: Basic $(echo -n admin:admin | base64)" \
    -H "Content-Type: application/json" \
    -XPUT "https://es-opendistro-node1:9200/_opendistro/_alerting/monitors/<monitor_id>/_acknowledge/alerts?pretty" \
    -d '
{
  "alerts": ["bn0_PmgBoCvkhulGF2K8"]
}'
```


### Referencias

* Documentación oficial sobre [Opendistro Alerting](https://opendistro.github.io/for-elasticsearch-docs/docs/alerting/).
* Levantar un Elasticsearch Opendistro con Docker Compose para usar el modulo de Alerting, en el [post anterior](https://ilittleangel.github.io/elk/elastic/opendistro/docker/docker-compose/ssl/2020/02/01/elasticsearch-opendistro-part1.html)
* Crear un [Incoming Webhook de Slack](https://api.slack.com/messaging/webhooks)
