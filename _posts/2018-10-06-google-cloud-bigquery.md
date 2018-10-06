# Google Cloud - Big Query

## API Java

If you plan to use a service account with client library code, you need to set an environment variable:

```sh
export GOOGLE_APPLICATION_CREDENTIALS=~/development/googlecloud-bigquery/src/main/resources/google-analytics-9ca2e8444354.json
```

Now, we can run a application without specify explicitly credentials

```sh
java -cp ~/development/googlecloud-bigquery/build/libs/googlecloud-bigquery-1.0-SNAPSHOT.jar \
     -Dhttp.proxyHost=proxy -Dhttp.proxyPort=8080 \
     -Dhttps.proxyHost=proxy -Dhttps.proxyPort=8080 \
     bigdata.googlecloud.bigquery.SimpleApp \
     google-analytics 112233445 ga_sessions_intraday_20180103
```

On the other hand, this option no need to export the env variable but need create credentials inside the code. So json file with credentials is an input argument

```sh
java -cp ~/development/googlecloud-bigquery/build/libs/googlecloud-bigquery-1.0-SNAPSHOT.jar \
     -Dhttp.proxyHost=proxy -Dhttp.proxyPort=8080 \
     -Dhttps.proxyHost=proxy -Dhttps.proxyPort=8080 \
     bigdata.googlecloud.bigquery.SimpleAppWithCred \
     ~/development/googlecloud-bigquery/src/main/resources/google-analytics-9ca2e8444354.json google-analytics 112233445 ga_sessions_intraday_20180103
```



## API REST

* [CLI installation and running](#cli-installation-and-running)
* [Authorization](#authorization)
* [Metadata](#metadata)
* [General Endpoints](#general-endpoints)
* [Testing](#testing)
    * [Tables `list`](#tables-list)
    * [Tables `get`](#tables-get)
    * [Tabledata `list`](#tabledata-list)
    * [Jobs `query`](#jobs-query)
    * [Jobs `get` (job status)](#jobs-get-job-status)
    * [Jobs `getQueryResults`](#jobs-getqueryresults)
    * [Jobs `insert` (extract data)](#jobs-insert-extract-data)
    * [Jobs `list`](#jobs-list)
* [Example of workflow](#example-of-workflow)



### CLI installation and running

##### Download gcloud
```sh
wget https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-182.0.0-linux-x86_64.tar.gz
```

##### Initialize gcloud

Add to `.bashrc`

```sh
gcloud init
```

### Authorization

###### Export env variable
```sh
export GOOGLE_APPLICATION_CREDENTIALS=~/development/googlecloud-bigquery/src/main/resources/google-analytics-9ca2e8444354.json
```

###### Create Access Token
```sh
ACCESS_TOKEN="$(gcloud auth application-default print-access-token)"
```

### Metadata

| ProjectId        | DatasetId | TableId                       |
|------------------|-----------|-------------------------------|
| google-analytics | 112233445 | ga_sessions_intraday_yyyymmdd |


### General Endpoints

###### List of main methods
```js
GET  https://www.googleapis.com/bigquery/v2/projects/projectId/datasets
GET  https://www.googleapis.com/bigquery/v2/projects/projectId/datasets/datasetId
GET  https://www.googleapis.com/bigquery/v2/projects/projectId/datasets/datasetId/tables/tableId
GET  https://www.googleapis.com/bigquery/v2/projects/projectId/datasets/datasetId/tables
GET  https://www.googleapis.com/bigquery/v2/projects/projectId/datasets/datasetId/tables/tableId/data
GET  https://www.googleapis.com/bigquery/v2/projects/projectId/jobs
POST https://www.googleapis.com/upload/bigquery/v2/projects/projectId/jobs
GET  https://www.googleapis.com/bigquery/v2/projects/projectId/jobs/jobId
POST https://www.googleapis.com/bigquery/v2/projects/projectId/jobs/jobId/cancel
POST https://www.googleapis.com/bigquery/v2/projects/projectId/queries
GET  https://www.googleapis.com/bigquery/v2/projects/projectId/queries/jobId
```


###### Examples
```js
GET  https://www.googleapis.com/bigquery/v2/projects/google-analytics/datasets/112233445/tables/ga_sessions_intraday_20171212
GET  https://www.googleapis.com/bigquery/v2/projects/google-analytics/datasets/112233445/tables/ga_sessions_intraday_20180103/data
POST https://www.googleapis.com/bigquery/v2/projects/google-analytics/queries
GET  https://www.googleapis.com/bigquery/v2/projects/google-analytics/queries/job_gRaHSTeIJCmIZzz4bctj0zsuffxc
```


###### Google reference

* [Datasets: `get`](https://cloud.google.com/bigquery/docs/reference/rest/v2/datasets/get)
* [Datasets: `list`](https://cloud.google.com/bigquery/docs/reference/rest/v2/datasets/list)
* [Tables: `get`](https://cloud.google.com/bigquery/docs/reference/rest/v2/tables/get)
* [Tables: `list`](https://cloud.google.com/bigquery/docs/reference/rest/v2/tables/list)
* [Tabledata: `list`](https://cloud.google.com/bigquery/docs/reference/rest/v2/tabledata/list)
* [Jobs: `query`](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs/query)
* [Jobs: `getQueryResults`](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs/getQueryResults)
* [Jobs: `get`](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs/get)
* [Jobs: `list`](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs/list)
* [Jobs: `insert`](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs/insert)



### Testing

##### Tables `list`
```sh
curl -X GET https://www.googleapis.com/bigquery/v2/projects/google-analytics/datasets/112233445/tables \
     -H "Authorization: Bearer $ACCESS_TOKEN"
```

##### Tables `get`
```sh
curl -X GET https://www.googleapis.com/bigquery/v2/projects/google-analytics/datasets/112233445/tables/ga_sessions_intraday_20180103 \
     -H "Authorization: Bearer $ACCESS_TOKEN"
```

##### Tabledata `list`
```sh
curl -X GET https://www.googleapis.com/bigquery/v2/projects/google-analytics/datasets/112233445/tables/ga_sessions_intraday_20180103/data \
     -H "Authorization: Bearer $ACCESS_TOKEN"
```

##### Jobs `query`

We must to run *JOBS*. Jobs are asynchronous tasks such as running queries, loading data, and exporting data. `Query` method run a job and return a job ID.

*Request*: With hardcode json
```sh
curl -X POST https://www.googleapis.com/bigquery/v2/projects/google-analytics/queries \
     -d '{"query":"select * from ga_sessions_intraday_20180103","kind":"bigquery#queryRequest","maxResults":2,"defaultDataset":{"projectId":"google-analytics","datasetId":"112233445"},"dryRun":false}' \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $ACCESS_TOKEN"
```

*Request*: With file [`request-body-query.json`](https://10.113.99.120:8443/angel/googlecloud/src/develop/src/main/resources/request-body-query.json)
```sh
curl -X POST https://www.googleapis.com/bigquery/v2/projects/google-analytics/queries \
     -d @/home/angelrojo/development/googlecloud-bigquery/src/main/resources/request-body-query.json \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $ACCESS_TOKEN"
```

`request-body-query.json` looks like:
```js
{
  "query": "SELECT visitorId,visitNumber,visitId,visitStartTime,date,hits.time,hits.hour,hits.minute,hits.referer,hits.page.pagePath FROM ga_sessions_intraday_20180104 OMIT RECORD IF COUNT(hits.hour) < 2 LIMIT 1",
  "kind": "bigquery#queryRequest",
  "maxResults": 2,
  "defaultDataset": {
    "projectId": "google-analytics",
    "datasetId": "112233445"
  },
  "dryRun": false,
  "useLegacySql": true
}
```

*Response*: If we don't set `timeout` parameter probably the query does not return data, but it return a `jobId` to use in `getQueryResults` method.
```js
{
 "kind": "bigquery#queryResponse",
 "jobReference": {
  "projectId": "google-analytics",
  "jobId": "job_RvsOhR4xxWHk43eKShWcqm5a0srh"
 },
 "jobComplete": false
}
```


##### Jobs `get` (job status)
Get the status of a specific job

```sh
curl -X GET https://www.googleapis.com/bigquery/v2/projects/google-analytics/jobs/job_35axds8rFh5v6x_ztzwCNIxvIR9- \
     -H "Authorization: Bearer $ACCESS_TOKEN"
```

One of the output fields in response body is `status.state`. It can be **PENDING** state, **RUNNING** or **DONE**.

Response has fields with destination table in BigQuery where result data has been inserted:

```js
...
"destinationTable": {
    "projectId": "google-analytics",
    "datasetId": "_9dbcf0c4fbdc437de6df08d8d7d353cd7888b31c",
    "tableId": "anonf7bb7b98737fb464a850db7bf8d5a0e3eacdc6de"
}
...
```


##### Jobs `getQueryResults`
```sh
curl -X GET https://www.googleapis.com/bigquery/v2/projects/google-analytics/queries/job_RvsOhR4xxWHk43eKShWcqm5a0srh/timeoutMs/60000 \
     -H "Authorization: Bearer $ACCESS_TOKEN" 
```

Response looks like this [getQueryResults json body](https://10.113.99.120:8443/angel/googlecloud/src/develop/src/main/resources/response-body-getQueryResults.json)


##### Jobs `insert` (extract data)

To extract data about specific JobId we must to use Google Cloud Storage.   

*Request*: With file [`request-body-insert.json`](https://10.113.99.120:8443/angel/googlecloud/src/develop/src/main/resources/request-body-insert.json)
```sh
curl -X POST https://www.googleapis.com/bigquery/v2/projects/google-analytics/jobs \
     -d @/home/angelrojo/development/googlecloud-bigquery/src/main/resources/request-body-insert.json \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $ACCESS_TOKEN"
```

`request-body-insert.json` looks like:
```js
{
  "jobReference": {
    "projectId": "google-analytics",
    "jobId": "custom-jobId-csv-20170104_1200"
  },
  "configuration": {
    "extract": {
      "sourceTable": {
        "projectId": "google-analytics",
        "datasetId": "_9dbcf0c4fbdc437de6df08d8d7d353cd7888b31c",
        "tableId":   "anonf7bb7b98737fb464a850db7bf8d5a0e3eacdc6de"
      },
      "destinationUris": ["gs://google-analytics/extraction-0001-ga_sessions_intraday_20180104.csv"],
      "destinationFormat": "CSV",
      "compression": "NONE"
    }
  }
}
```


##### Jobs `list`
```sh
curl -X GET https://www.googleapis.com/bigquery/v2/projects/google-analytics/jobs \
     -H "Authorization: Bearer $ACCESS_TOKEN"
```


### Example of workflow

1. Export de la variable `GOOGLE_APPLICATION_CREDENTIALS` contiene el path del json con credenciales
2. Obtener `ACCESS_TOKEN`
3. Llamar a `query` recoger el jobID -> `job_35axds8rFh5v6x_ztzwCNIxvIR9-`
4. Llamar a `get` para el job status y recoger del body response los campos `jsodatasetId` y `tableId` del objeto `destinationTable`
5. Llamar a `insert` para hacer el extract, pasandole el `jobId` custom, el `datasetId` y `tableId` anteriores. Esto escribe un fichero en el formato indicado, en este caso CSV, en un bucket de GCS (Google Cloud Storage). Hay que parametrizar el nombre del fichero csv para que sea distinto por cada llamada a BigQuery
6. Llamar al API Rest de Google Cloud Storage

