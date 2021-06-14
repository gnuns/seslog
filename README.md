# seslog

`seslog` is nginx syslog server.
It collects nginx's access logs and write them to ClickHouse DB.

## Make
```bash
make
```

## Run
```bash
build/seslog-server -logtostderr=true
```
Run `build/seslog-server --help` to see all options

## Init table

```sql
CREATE DATABASE IF NOT EXISTS seslog;

CREATE TABLE IF NOT EXISTS seslog.access_log
(
  nginx_event_date Date DEFAULT toDate(nginx_event_time),
  nginx_tag String,
  nginx_event_time DateTime DEFAULT now(),
  nginx_hostname String,
  nginx_ip_uint32 UInt32,

  zonename String,
  zoneoffset Int32,

  body_bytes_sent UInt64,
  connections_active UInt16,
  connections_reading UInt16,
  connections_waiting UInt16,
  connections_writing UInt16,
  content_length UInt64,

  http_scheme String,
  http_domain String,
  http_path String,
  http_arg_keys Array(String),
  http_arg_vals Array(Array(String)),

  http_referer_scheme String,
  http_referer_domain String,
  http_referer_path String,
  http_referer_arg_keys Array(String),
  http_referer_arg_vals Array(Array(String)),

  http_location_scheme String,
  http_location_domain String,
  http_location_path String,
  http_location_arg_keys Array(String),
  http_location_arg_vals Array(Array(String)),

  ua_family String,
  ua_major String,
  ua_minor String,
  ua_patch String,
  ua_os_family String,
  ua_os_major String,
  ua_os_minor String,
  ua_os_patch String,
  ua_os_patchminor String,
  ua_device_family String,

  http_x_forwarded_for String,
  remote_addr_uint32 UInt32,
  request_method String,
  request_time Float64,

  status UInt16,
  tcpinfo_rtt UInt64,
  tcpinfo_rttvar UInt64,

  upstream_response_length UInt64,
  upstream_response_time Float64,
  upstream_status UInt16,
  uri String
) ENGINE = MergeTree PARTITION BY toYYYYMM(nginx_event_date) ORDER BY (nginx_event_date, nginx_tag, nginx_event_time, nginx_hostname);

```


## Install as systemd service
```bash
sudo make install
```

## Install to nginx
Add new `log_format` to your nginx config (http section)
```
log_format seslog_format '$body_bytes_sent\t$connections_active\t$connections_reading\t$connections_waiting\t$connections_writing\t$content_length\t$http_host\t$http_referer\t$http_user_agent\t$http_x_forwarded_for\t$remote_addr\t$request_method\t$request_time\t$request_uri\t$scheme\t$status\t$tcpinfo_rtt\t$tcpinfo_rttvar\t$time_local\t$upstream_cache_status\t$upstream_response_length\t$upstream_response_time\t$upstream_status\t$uri\t$sent_http_location';
```

Then add access_log (preferred section)  
```
access_log syslog:server=<YOUR_SESLOG_IP>:5514,tag=<YOUR_PROJECT_NAME> seslog_format;
```
For example:
```
access_log syslog:server=127.0.0.1:5514,tag=php_admin_panel seslog_format if=$sesloggable;
```

## Tips
Please use `$loggable` (or anything like that) variable to avoid useless logging

e.g. (http context):
```
map $request_uri $sesloggable {
    default                                             1;
    ~*\.(ico|css|js|gif|jpg|jpeg|png|svg|woff|ttf|eot)$ 0;
}
```

## SELECT data
Select TOP-30 Referrer Domains
```clickhouse
SELECT
    http_referer_domain,
    count() AS cnt
FROM access_log 
WHERE nginx_event_date = today()
GROUP BY http_referer_domain
ORDER BY cnt DESC
LIMIT 30
```