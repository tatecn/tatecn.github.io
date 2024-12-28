---
title: use-the-logstash-jdbc-input-plugin-to-ingest-data-into-aws-opensearch
date: 2024-12-20 19:06:00 +0800
categories: [AWS, OpenSearch]
tags: [logstash,jdbc input plugin,logstash-output-opensearch]
description: How to index document for geo_shape field using logstash jdbc input.
---

## 0. Background
It is common to synchronize data in real-time from RDBMS (such as MySQL) to OpenSearch(or Elasticsearch) using Logstash. This blog documents the process and provides key considerations, particularly for the `geo_shape` field.

## 1. Prepare Your Database
Ensure your database table contains geometric data in a supported format, such as GeoJSON. I will use `geo_shape_demo` as table name and index name. Example:

```sql
CREATE TABLE `geo_shape_demo` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `jd_no` varchar(32) NOT NULL,
  `job_geo_bounds` varchar(500) DEFAULT NULL,
  `update_time` timestamp(3) DEFAULT current_timestamp(3) ON UPDATE current_timestamp(3),
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_jd_no` (`jd_no`)
) ENGINE=InnoDB AUTO_INCREMENT=10 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci

insert into geo_shape_demo (jd_no,job_geo_bounds) values ('1814625742831620096','{
  "coordinates": [
    [
      [
        -123.63249969482422,
        38.86429977416992
      ],
      [
        -123.63249969482422,
        36.892974853515625
      ],
      [
        -121.20817565917969,
        36.892974853515625
      ],
      [
        -121.20817565917969,
        38.86429977416992
      ],
      [
        -123.63249969482422,
        38.86429977416992
      ]
    ]
  ],
  "type": "polygon"
}');
```

## 2. Define the Elasticsearch Index Mapping

```json
PUT /geo_shape_demo
{
  "mappings": {
    "properties": {
      "jdNo": {
        "type": "keyword"
      },
      "jobGeoBounds": {
        "type": "geo_shape"
      },
      "updateTime": {
        "type": "date"
      }
    }
  }
}
```
## 3. Download logstash with logstash-output-opensearch plugin
If you use AWS OpenSearch service you could download 

[https://artifacts.opensearch.org/logstash/logstash-oss-with-opensearch-output-plugin-8.9.0-linux-x64.tar.gz](https://artifacts.opensearch.org/logstash/logstash-oss-with-opensearch-output-plugin-8.9.0-linux-x64.tar.gz)
from [this page](https://opensearch.org/downloads.html).

**Note:**
Please select the correct version of the Logstash `logstash-output-opensearch` plugin that corresponds to your `OpenSearch` version.

You can also install this plugin separately if you already have a Logstash service.

## 4. Configure Logstash Pipeline
Set up your Logstash configuration file (pipeline.conf) to read from the database and index into OpenSearch.

```ruby
input {
   jdbc {
      jdbc_driver_library => "/path/to/your/jdbc/driver.jar"
      jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
      jdbc_connection_string => "jdbc:mysql://your_database_url"
      jdbc_user => "your_username"
      jdbc_password => "your_password"
      statement => "SELECT 
      jd_no AS jdNo,
      job_geo_bounds AS jobGeoBounds,
      FLOOR(UNIX_TIMESTAMP(update_time)*1000) AS updateTime
      UNIX_TIMESTAMP(update_time) AS unix_ts_in_secs 
      FROM geo_shape_demo 
      WHERE UNIX_TIMESTAMP(update_time) > :sql_last_value 
      ORDER BY update_time ASC "
      record_last_run => true
      last_run_metadata_path => "/your_last_run_metadata_path"
      tracking_column => "unix_ts_in_secs"
      use_column_value => true
      tracking_column_type => "numeric"
      schedule => "*/5 * * * * *"
      clean_run => true
      lowercase_column_names => false
	}
}

output {
      opensearch  {
         hosts => "https://your_host:your_port"
         user => "your_username"
         password => "your_password"
         index => "geo_shape_demo"
         document_id => "%{jdNo}"
         ssl_certificate_verification => false
         ecs_compatibility => disabled
      }
      stdout { codec =>  "rubydebug"}
}
```
## 5. Run Logstash
Start Logstash with the configuration file:
```bash
bin/logstash -f pipeline.conf
```

## 6. Tips
### 6.1 parse_exception
Remember to add a `JSON` filter plugin; otherwise, you may encounter a `parse_exception`.

```ruby
filter {
  mutate {
    remove_field => ["id", "@version", "unix_ts_in_secs"]
  }
  
  json {
    source => "jobGeoBounds"
    target => "jobGeoBounds"
  }
}
```
`parse_exception` info like this:
>"status"=>400, <br>
>"error"=>{"type"=>"mapper_parsing_exception", <br>
>"reason"=>"failed to parse field [xxx] of type [geo_shape]", <br>
>"caused_by"=>{ <br>
>  "type"=>"parse_exception", <br>
>  "reason"=>"expected **word** but found: **'{'**" <br>
>}

### 6.2 timestamp field
Typically, you may want to use a `long` number to store a `timestamp` field in both the database and OpenSearch, as `millisecond` precision is sometimes required.
In MySQL/MariaDB, the `UNIX_TIMESTAMP` function returns a floating-point number (instead of the expected integer) when the input is a timestamp field with millisecond precision. Since timestamp fields are stored in scientific notation within the index, you need to use the `FLOOR` function to convert the float to an integer.

**Reference links:**
<br>
[https://www.elastic.co/guide/en/cloud/current/ec-getting-started-search-use-cases-db-logstash.html](https://www.elastic.co/guide/en/cloud/current/ec-getting-started-search-use-cases-db-logstash.html)
<br>
[https://opensearch.org/docs/latest/tools/logstash/ship-to-opensearch/](https://opensearch.org/docs/latest/tools/logstash/ship-to-opensearch/)
