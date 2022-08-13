# starlette_exporter

## Prometheus exporter for Starlette and FastAPI

starlette_exporter collects basic metrics for Starlette and FastAPI based applications:

* starlette_requests_total: a counter representing the total requests
* starlette_request_duration_seconds: a histogram representing the distribution of request response times
* starlette_requests_in_progress: a gauge that keeps track of how many concurrent requests are being processed

Metrics include labels for the HTTP method, the path, and the response status code.

```
starlette_requests_total{method="GET",path="/",status_code="200"} 1.0
starlette_request_duration_seconds_bucket{le="0.01",method="GET",path="/",status_code="200"} 1.0
```

Use the HTTP handler `handle_metrics` at path `/metrics` to expose a metrics endpoint to Prometheus.

## Table of Contents

- [starlette\_exporter](#starlette_exporter)
  - [Prometheus exporter for Starlette and FastAPI](#prometheus-exporter-for-starlette-and-fastapi)
  - [Table of Contents](#table-of-contents)
  - [Usage](#usage)
    - [Starlette](#starlette)
    - [FastAPI](#fastapi)
  - [Options](#options)
  - [Custom Metrics](#custom-metrics)
      - [Example:](#example)
  - [Multiprocess mode (gunicorn deployments)](#multiprocess-mode-gunicorn-deployments)
  - [Developing](#developing)
  - [License](#license)
  - [Dependencies](#dependencies)
  - [Credits](#credits)

## Usage

```sh
pip install starlette_exporter
```

### Starlette

```python
from starlette.applications import Starlette
from starlette_exporter import PrometheusMiddleware, handle_metrics

app = Starlette()
app.add_middleware(PrometheusMiddleware)
app.add_route("/metrics", handle_metrics)

...
```

### FastAPI

```python
from fastapi import FastAPI
from starlette_exporter import PrometheusMiddleware, handle_metrics

app = FastAPI()
app.add_middleware(PrometheusMiddleware)
app.add_route("/metrics", handle_metrics)

...
```

## Options

`app_name`: Sets the value of the `app_name` label for exported metrics (default: `starlette`).

`prefix`: Sets the prefix of the exported metric names (default: `starlette`).

`labels`: Optional dict containing default labels that will be added to all metrics. The values can be either a static value or a callback function that
retrieves a value from the `Request` object. [See below](#labels) for examples.

`group_paths`: setting this to `True` will populate the path label using named parameters (if any) in the router path, e.g. `/api/v1/items/{item_id}`.  This will group requests together by endpoint (regardless of the value of `item_id`). This option may come with a performance hit for larger routers. Default is `False`, which will result in separate metrics for different URLs (e.g., `/api/v1/items/42`, `/api/v1/items/43`, etc.).

`filter_unhandled_paths`: setting this to `True` will cause the middleware to ignore requests with unhandled paths (in other words, 404 errors). This helps prevent filling up the metrics with 404 errors and/or intentially bad requests. Default is `False`.

`buckets`: accepts an optional list of numbers to use as histogram buckets. The default value is `None`, which will cause the library to fall back on the Prometheus defaults (currently `[0.01, 0.025, 0.05, 0.075, 0.1, 0.25, 0.5, 0.75, 1.0, 2.5, 5.0, 7.5, 10.0]`).

`skip_paths`: accepts an optional list of paths that will not collect metrics. The default value is `None`, which will cause the library to collect metrics on every requested path. This option is useful to avoid collecting metrics on health check, readiness or liveness probe endpoints.

`always_use_int_status`: accepts a boolean. The default value is False. If set to True the libary will attempt to convert the `status_code` value to an integer. If the conversion fails it will log a warning and use the initial value. This is useful if you use [`http.HTTStatus`](https://docs.python.org/3/library/http.html#http.HTTPStatus) in your code but want your metrics to emit only a integer status code.

`optional_metrics`: a list of pre-defined metrics that can be optionally added to the default metrics. The following optional metrics are available:
  * `response_body_size`: a counter that tracks the size of response bodies for each endpoint

For optional metric examples, [see below](#optional-metrics).

Full example:
```python
app.add_middleware(
  PrometheusMiddleware,
  app_name="hello_world",
  prefix='myapp',
  labels={
      "server": os.getenv("HOSTNAME"),
      "host": lambda r: r.headers.get("host")
  }),
  group_paths=True,
  buckets=[0.1, 0.25, 0.5],
  skip_paths=['/health'],
  always_use_int_status=False,
```

## Labels

**Warning: this feature is experimental.**

Metrics have built-in default labels including `app_name`, `method`, `path`, and `status_code`.  Additional default labels can be
added by passing a dictionary to the `labels` arg to `PrometheusMiddleware`.  Each label's value can be either a static
value or, optionally, a callback function.

If a callback function is used, it will receive the Request instance as its argument.

```python
app.add_middleware(
  PrometheusMiddleware,
  labels={
     "host": lambda r: r.headers.get("host")
     "service": "api"
    }
```

This functionality is experimental.  Exceptions may be raised if the label name, value or callback function is malformed. 
Ensure that label names follow [Prometheus naming conventions](https://prometheus.io/docs/practices/naming/).

### Label helpers

`from_header(key: string, allowed_values: Optional[Iterable])`:  a convenience function for using a header value as a label.
`allowed_values` allows you to supply a list of allowed values. If supplied, header values not in the list will result in
an empty string being returned.  This allows you to constrain the label values, reducing the risk of excessive cardinality.

```python
from starlette_exporter import PrometheusMiddleware, from_header

app.add_middleware(
  PrometheusMiddleware,
  labels={
      "host": from_header("X-User", allowed_values=("frank", "estelle"))
    }
```



## Optional metrics

Optional metrics are pre-defined metrics that can be added to the default metrics.

  * `response_body_size`: the size of response bodies returned, in bytes
  * `request_body_size`: the size of request bodies received, in bytes

#### Example:

```python
from fastapi import FastAPI
from starlette_exporter import PrometheusMiddleware, handle_metrics
from starlette_exporter.optional_metrics import response_body_size, request_body_size

app = FastAPI()
app.add_middleware(PrometheusMiddleware, optional_metrics=[response_body_size, request_body_size])
```

## Custom Metrics

starlette_exporter will export all the prometheus metrics from the process, so custom metrics can be created by using the prometheus_client API.

#### Example:

```python
from prometheus_client import Counter
from starlette.responses import RedirectResponse

REDIRECT_COUNT = Counter("redirect_total", "Count of redirects", ["redirected_from"])

async def some_view(request):
    REDIRECT_COUNT.labels("some_view").inc()
    return RedirectResponse(url="https://example.com", status_code=302)
```

The new metric will now be included in the the `/metrics` endpoint output:

```
...
redirect_total{redirected_from="some_view"} 2.0
...
```

## Multiprocess mode (gunicorn deployments)

Running starlette_exporter in a multiprocess deployment (e.g. with gunicorn) will need the `PROMETHEUS_MULTIPROC_DIR` env variable set, as well as extra gunicorn config.

For more information, see the [Prometheus Python client documentation](https://github.com/prometheus/client_python#multiprocess-mode-eg-gunicorn).

## Developing

This package supports Python 3.6+.

```sh
git clone https://github.com/stephenhillier/starlette_exporter
cd starlette_exporter
pytest tests
```

## License

Code released under the [Apache License, Version 2.0](./LICENSE).


## Dependencies

https://github.com/prometheus/client_python

https://github.com/encode/starlette

## Credits

Starlette - https://github.com/encode/starlette

FastAPI - https://github.com/tiangolo/fastapi

Flask exporter - https://github.com/rycus86/prometheus_flask_exporter

Alternate Starlette exporter - https://github.com/perdy/starlette-prometheus
