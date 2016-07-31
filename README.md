# Prometheus metric library for Nginx

This is a Lua library that can be used with Nginx to keep track of metrics and
expose them on a separate web page to be pulled by
[Prometheus](https://prometheus.io).

## Quick start guide

You would need to install nginx package with lua support (`nginx-extras` on
Debian) and make `prometheus.lua` available in your `LUA_PATH` (or just point
`lua_package_path` to a directory with this git repo).

To track request latency broken down by HTTP host and request count broken
down by host and status, add the following to `nginx.conf`:

```
lua_shared_dict prometheus_metrics 10M;
lua_package_path "/path/to/nginx-lua-prometheus/?.lua";
init_by_lua '
  prometheus = require("prometheus").init("prometheus_metrics")
  metric_requests = prometheus:counter(
    "nginx_http_requests_total", "Number of HTTP requests", {"host", "status"})
  metric_latency = prometheus:histogram(
    "nginx_http_request_duration_seconds", "HTTP request latency", {"host"})
  metric_connections_current = prometheus:gauge(
    "nginx_http_connections_current", "Number of HTTP connections currently processed", {"state"})
';
log_by_lua '
  local host = ngx.var.host:gsub("^www.", "")
  metric_requests:inc(1, {host, ngx.var.status})
  metric_latency:observe(ngx.now() - ngx.req.start_time(), {host})
  metric_connections_current:set(tonumber(ngx.var.connections_active), {"active"})
  metric_connections_current:set(tonumber(ngx.var.connections_reading), {"reading"})
  metric_connections_current:set(tonumber(ngx.var.connections_waiting), {"waiting"})
  metric_connections_current:set(tonumber(ngx.var.connections_writing), {"writing"})
';
```

This:
* configures a shared dictionary for your metrics called `prometheus_metrics`
  with a 10MB size limit;
* registers a counter called `nginx_http_requests_total` with two labels:
  `host` and `status`;
* registers a histogram called `nginx_http_request_duration_seconds` with one
  label `host`;
* on each HTTP request measures its latency, recording it in the histogram and
  increments the counter, setting current HTTP host as `host` label and
  HTTP status code as `status` label.
* registers a gauge called `nginx_http_connections_current` with one label `state`;

Last step is to configure a separate server that will expose the metrics.
Please make sure to only make it reachable from your Prometheus server:

```
server {
  listen 9145;
  allow 192.168.0.0/16;
  deny all;
  location /metrics {
    content_by_lua 'prometheus:collect()';
  }
}
```

Metrics will be available at `http://your.nginx:9145/metrics`.

**Note**: using HTTP host as a metric label value on servers that have many
virtual hosts has potential performance implications. Please read the caveats
section below for more information.

## API reference

### init()

**syntax:** require("prometheus").init(*dict_name*, [*prefix*])

Initializes the module. This should be called once from the
[init_by_lua](https://github.com/openresty/lua-nginx-module#init_by_lua)
section in nginx configuration.

* `dict_name` is the name of the nginx shared dictionary which will be used to
  store all metrics. Defaults to `prometheus_metrics` if not specified.
* `prefix` is an optional string which will be prepended to metric names on output


Returns a `prometheus` object that should be used to register metrics.

Example:
```
init_by_lua '
  prometheus = require("prometheus").init("prometheus_metrics")
';
```

### prometheus:counter()

**syntax:** prometheus:counter(*name*, *description*, *label_names*)

Registers a counter. Should be called once from the
[init_by_lua](https://github.com/openresty/lua-nginx-module#init_by_lua)
section.

* `name` is the name of the metric.
* `description` is the text description that will be presented to Prometheus
  along with the metric. Optional (pass `nil` if you still need to define
  label names).
* `label_names` is an array of label names for the metric. Optional.

[Naming section](https://prometheus.io/docs/practices/naming/) of Prometheus
documentation provides good guidelines on choosing metric and label names.

Returns a `counter` object that can later be incremented.

Example:
```
init_by_lua '
  prometheus = require("prometheus").init("prometheus_metrics")
  metric_bytes = prometheus:counter(
    "nginx_http_request_size_bytes", "Total size of incoming requests")
  metric_requests = prometheus:counter(
    "nginx_http_requests_total", "Number of HTTP requests", {"host", "status"})
';
```

### prometheus:histogram()

**syntax:** prometheus:histogram(*name*, *description*, *label_names*,
  *buckets*)

Registers a histogram. Should be called once from the
[init_by_lua](https://github.com/openresty/lua-nginx-module#init_by_lua)
section.

* `name` is the name of the metric.
* `description` is the text description. Optional.
* `label_names` is an array of label names for the metric. Optional.
* `buckets` is an array of numbers defining bucket boundaries. Optional,
  defaults to 20 latency buckets covering a range from 5ms to 10s (in seconds).

Returns a `histogram` object that can later be used to record samples.

Example:
```
init_by_lua '
  prometheus = require("prometheus").init("prometheus_metrics")
  metric_latency = prometheus:histogram(
    "nginx_http_request_duration_seconds", "HTTP request latency", {"host"})
  metric_response_sizes = prometheus:counter(
    "nginx_http_response_size_bytes", "Size of HTTP responses", nil,
    {10,100,1000,10000,100000,1000000})
';
```

### prometheus:collect()

**syntax:** prometheus:collect()

Presents all metrics in a text format compatible with Prometheus. This should be
called in
[content_by_lua](https://github.com/openresty/lua-nginx-module#content_by_lua)
to expose the metrics on a separate HTTP page.

Example:
```
location /metrics {
  content_by_lua 'prometheus:collect()';
  allow 192.168.0.0/16;
  deny all;
}
```

### counter:inc()

**syntax:** counter:inc(*value*, *label_values*)

Increments a previously registered counter. This is usually called from
[log_by_lua](https://github.com/openresty/lua-nginx-module#log_by_lua)
globally or per server/location.

* `value` is a value that should be added to the counter. Defaults to 1.
* `label_values` is an array of label values. The number of values should
  match the number of label names defined when the counter was registered
  using `prometheus:counter()`. No label values should be provided for
  counters with no labels.

Example:
```
log_by_lua '
  metric_bytes:inc(tonumber(ngx.var.request_length))
  metric_requests:inc(1, {ngx.var.host, ngx.var.status})
';
```

### histogram:observe()

**syntax:** histogram:observe(*value*, *label_values*)

Records a value in a previously registered histogram. Usually called from
[log_by_lua](https://github.com/openresty/lua-nginx-module#log_by_lua)
globally or per server/location.

* `value` is a value that should be recorded. Required.
* `label_values` is an array of label values. The number of values should
  match the number of label names defined when the histogram was registered
  using `prometheus:histogram()`. No label values should be provided for
  histograms with no labels.

Example:
```
log_by_lua '
  metric_latency:observe(ngx.now() - ngx.req.start_time(), {ngx.var.host})
  metric_response_sizes:observe(tonumber(ngx.var.bytes_sent))
';
```

### Built-in metrics

The module increments the `nginx_metric_errors_total` metric if it encounters
an error (for example, when `lua_shared_dict` becomes full). You might want
to configure an alert on that metric.

## Caveats

Please keep in mind that all metrics stored by this library are kept in a
single shared dictionary (`lua_shared_dict`). While exposing metrics the module
has to list all dictionary keys, which has serious performance implications for
dictionaries with large number of keys (in this case this means large number
of metrics OR metrics with high label cardinality). Listing the keys has to
lock the dictionary, which blocks all threads that try to access it (i.e.
potentially all nginx worker threads).

There is no elegant solution to this issue (besides keeping metrics in a
separate storage system external to nginx), so for latency-critical servers you
might want to keep the number of metrics (and distinct metric label values) to
a minimum.

## Development

### Install dependencies for testing

- `luarocks install luacheck`
- `luarocks install luaunit`

### Run tests

- `luacheck --globals ngx -- prometheus.lua`
- `lua prometheus_test.lua`

## License

Licensed under MIT license.
