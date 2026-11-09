# Elasticsearch REST API -- Kibana Console and Command Line

## 1. What is Elasticsearch REST API?

Elasticsearch provides REST APIs to manage, store, search, update, and
delete data using HTTP requests.

You can execute these APIs in two common ways:

1.  **Kibana Dev Tools Console** -- run API requests from the Kibana
    interface.
2.  **Linux Command Line** -- use `curl` to send API requests directly
    to Elasticsearch.

## 2. Accessing the Kibana Console

1.  Log in to Kibana.
2.  Open **Management → Dev Tools** (menu names may vary by Kibana
    version).
3.  Select **Console**.
4.  Enter an API request and click the run button.

### Check Elasticsearch health

``` http
GET _cluster/health
```

### Check cluster nodes

``` http
GET _cat/nodes?v
```

### List all indices

``` http
GET _cat/indices?v
```

### Create an index

``` http
PUT employees
```

### Insert a document

``` http
POST employees/_doc/1
{
  "name": "Rahul",
  "department": "DevOps",
  "experience": 5
}
```

### Get a document

``` http
GET employees/_doc/1
```

### Search documents

``` http
GET employees/_search
{
  "query": {
    "match_all": {}
  }
}
```

### Update a document

``` http
POST employees/_update/1
{
  "doc": {
    "department": "Cloud"
  }
}
```

### Delete a document

``` http
DELETE employees/_doc/1
```

### Delete an index

``` http
DELETE employees
```

## 3. Accessing Elasticsearch from the Linux Command Line

Use `curl` to send REST API requests from a terminal. The examples below
assume Elasticsearch is running on `localhost:9200`.

### Check Elasticsearch status

``` bash
curl http://localhost:9200
```

### Check cluster health

``` bash
curl "http://localhost:9200/_cluster/health?pretty"
```

### Check cluster nodes

``` bash
curl "http://localhost:9200/_cat/nodes?v"
```

### List all indices

``` bash
curl "http://localhost:9200/_cat/indices?v"
```

### Create an index

``` bash
curl -X PUT "http://localhost:9200/employees?pretty"
```

### Insert a document

``` bash
curl -X POST "http://localhost:9200/employees/_doc/1?pretty" \
-H "Content-Type: application/json" \
-d '{
  "name": "Rahul",
  "department": "DevOps",
  "experience": 5
}'
```

### Get a document

``` bash
curl -X GET "http://localhost:9200/employees/_doc/1?pretty"
```

### Search documents

``` bash
curl -X GET "http://localhost:9200/employees/_search?pretty"
```

### Update a document

``` bash
curl -X POST "http://localhost:9200/employees/_update/1?pretty" \
-H "Content-Type: application/json" \
-d '{
  "doc": {
    "department": "Cloud"
  }
}'
```

### Delete a document

``` bash
curl -X DELETE "http://localhost:9200/employees/_doc/1?pretty"
```

### Delete an index

``` bash
curl -X DELETE "http://localhost:9200/employees?pretty"
```

## 4. Common HTTP Methods

  Method   Purpose
  -------- ----------------------------------------
  GET      Retrieve data
  POST     Create a document or perform an update
  PUT      Create or replace a resource
  DELETE   Delete a document or index

## 5. Important Points

-   Elasticsearch default HTTP port: `9200`
-   Kibana default port: `5601`
-   Kibana Console lets you execute REST API requests without writing
    `curl` commands.
-   `curl` is commonly used to access REST APIs from Linux.
-   `?pretty` formats JSON output for easier reading.
-   In `curl`, `-X` specifies the HTTP method, `-H` specifies a header,
    and `-d` sends request data.
-   Elasticsearch stores data as JSON documents inside indices.

## 6. Kibana Console vs Command Line

  -----------------------------------------------------------------------
  Kibana Dev Tools Console            Linux Command Line
  ----------------------------------- -----------------------------------
  Runs inside Kibana                  Runs in a terminal

  Uses Elasticsearch API syntax       Uses `curl` with an HTTP URL
  directly                            

  Easy for testing and learning APIs  Useful for scripts and automation

  No need to write the full host URL  Requires the Elasticsearch URL and
  for the configured cluster          any required authentication
  -----------------------------------------------------------------------

## 7. Interview Answer

**Q: How do you access Elasticsearch REST APIs?**

I can access Elasticsearch REST APIs in two ways. First, I use Kibana
Dev Tools Console to execute API requests directly. Second, I use the
`curl` command from a Linux terminal. With GET, POST, PUT, and DELETE
methods, I can check cluster health, list indices, create indices,
insert and search documents, update records, and delete data.

## 8. Important Safety Note

These examples use HTTP and assume Elasticsearch is available locally
without authentication. In a secured environment, use the configured
HTTPS endpoint and required authentication, for example `-u` with `curl`
where appropriate. Avoid putting passwords directly in shell history.
Deleting a document or index is destructive, so verify the target before
running DELETE commands.
