# telemetry-subscribers
Common utilities for Tokio-based application telemetry, including tracing, logging, spans

This is a library for common telemetry functionality, especially subscribers for [Tokio tracing](https://github.com/tokio-rs/tracing)
libraries.  Here we simply package many common subscribers, such as writing trace data to Jaeger, distributed tracing,
common logs and metrics destinations, etc.  into a easy to configure common package.  There are also
some unique layers such as one to automatically create Prometheus latency histograms for spans.

We also purposely separate out logging levels from span creation.  This is often needed by production apps
as normally it is not desired to log at very high levels, but still desirable to gather sampled span data
all the way down to TRACE level spans.

Getting started is easy.  In your app:

```rust
  use telemetry_subscribers::TelemetryConfig;
  let (_guard, _handle) = TelemetryConfig::new("my_app").init();
```

It is important to retain the guard until the end of the program.  Assign it in the main fn and keep it,
for once it drops then log output will stop.

There is a builder API available: just do `TelemetryConfig::new()...` Another convenient initialization method
is `TelemetryConfig::new().with_env()` to populate the config from environment vars.

You can also run the example and see output in ANSI color:

    cargo run --example easy-init

## Features
- `jaeger` - this feature is enabled by default as it enables jaeger tracing
- `json` - uses the JSON feature from the [tracing crate](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/format/struct.Json.html), optional
- `tokio-console` - [Tokio-console](https://github.com/tokio-rs/console) subscriber, optional
- `chrome` - enables use of `chrome://tracing` to visualize output, optional

### Stdout vs file output

By default, logs (but not spans) are formatted for human readability and output to stdout, with key-value tags at the end of every line.
`RUST_LOG` can be configured for custom logging output, including filtering.

By setting `log_file` in the config, one can write log output to a daily-rotated file.

### Tracing and span output

Detailed span start and end logs can be generated in two ways:
* by defining the `span_log_output` config variable / `ENABLE_SPAN_LOGS` env var.
* by defining the `json_log_output` config variable / `ENABLE_JSON_LOGS` env var.  Note that this causes output to be in JSON format, which is not as human-readable, so it is not enabled by default.

JSON output can easily be fed to backends such as ElasticSearch for indexing, alerts, aggregation, and ysis.
It requires the `json` crate feature to be enabled.

### Jaeger (seeing distributed traces)

To see nested spans visualized with [Jaeger](https://www.jaegertracing.io), do the following:

1. Run this to get a local Jaeger container: `docker run -d -p6831:6831/udp -p6832:6832/udp -p16686:16686 jaegertracing/all-in-one:1.48`
2. Set `enable_jaeger` config setting to true or set `TOKIO_JAEGER` env var
3. Run your app
4. Browse to `http://localhost:16686/` and select the service you configured using `service_name`

NOTE: separate spans (which are not nested) are not connected as a single trace for now.
NOTE2: The jaegertracing container `latest` tag does not seem to contain `arm64` images for Apple silicon Macbooks.

Jaeger subscriber is enabled by default but is protected by the jaeger feature flag.  If you'd like to leave
out the Jaeger dependencies, you can turn off the default-features in your dependency:

    telemetry = { url = "...", default-features = false }

### Automatic Prometheus span latencies

Included in this library is a tracing-subscriber layer named `PrometheusSpanLatencyLayer`.  It will create
a Prometheus histogram to track latencies for every span in your app, which is super convenient for tracking
span performance in production apps.

Enabling this layer can only be done programmatically, by passing in a Prometheus registry to `TelemetryConfig`.

### Span levels vs log levels

What spans are included for Jaeger output, automatic span latencies, etc.?  These are controlled by
the `span_level` config attribute, or the `TOKIO_SPAN_LEVEL` environment variable.  Note that this is
separate from `RUST_LOG`, so that you can separately control the logging verbosity from the level of
spans that are to be recorded and traced.

Note that span levels for regular logging output are not affected by the span level config.

### Live async inspection / Tokio Console

[Tokio-console](https://github.com/tokio-rs/console) is an awesome CLI tool designed to analyze and help debug Rust apps using Tokio, in real time!  It relies on a special subscriber.

1. Build your app using a special flag: `RUSTFLAGS="--cfg tokio_unstable" cargo build`
2. Enable the `tokio-console` feature for this crate.
2. Set the `tokio_console` config setting when running your app (or set TOKIO_CONSOLE env var if using config `with_env()` method)
3. Clone the console repo and `cargo run` to launch the console

NOTE: setting tokio TRACE logs is NOT necessary.  It says that in the docs but there's no need to change Tokio logging levels at all.  The console subscriber has a special filter enabled taking care of that.

By default, Tokio console listens on port 6669.  To change this setting as well as other setting such as
the retention policy, please see the [configuration](https://docs.rs/console-subscriber/latest/console_subscriber/struct.Builder.html#configuration) guide.

### Custom panic hook

This library installs a custom panic hook which records a log (event) at ERROR level using the tracing
crate.  This allows span information from the panic to be properly recorded as well.

To exit the process on panic, set the `CRASH_ON_PANIC` environment variable.
