# Servers

The `server` package provides pre-configured HTTP and gRPC server implementations designed to work seamlessly with the `config` and `controls` packages. It establishes standard APIs, routing, middleware integration, and request handling capabilities suited for scalable backend services.

## HTTP Server

The HTTP server component is a wrapper around the standard `net/http.Server`, including best-practice defaults for timeouts and security.

### Key Features
- **Timeouts**: Pre-configured defaults for `ReadTimeout` (5s), `WriteTimeout` (10s), and `IdleTimeout` (120s).
- **TLS Configuration**: Supports TLS 1.2+ with a curated list of secure cipher suites and curve preferences.
- **Graceful Shutdown**: Integrates with the `controls` package to provide a standard `Stop` function that handles graceful shutdown via `srv.Shutdown(ctx)`.

### Controls Integration
- `NewServer(ctx, cfg, handler)`: Creates and returns a configured `*http.Server`.
- `Start(cfg, logger, srv)`: Returns a `controls.StartFunc` that handles both cleartext and TLS starting based on configuration.
- `Stop(logger, srv)`: Returns a `controls.StopFunc` for graceful shutdown.

## gRPC Server

The gRPC server component provides a wrapper around `google.golang.org/grpc.Server`.

### Key Features
- **Reflection**: Automatically registers the reflection service, making it easier to use tools like `grpcurl`.
- **Custom Options**: Supports passing through standard `grpc.ServerOption` during initialization.

### Controls Integration
- `NewServer(cfg, opt...)`: Creates and returns a configured `*grpc.Server` with reflection enabled.
- `Start(cfg, logger, srv)`: Returns a `controls.StartFunc` that listens on the configured port and starts the server.
- `Stop(logger, srv)`: Returns a `controls.StopFunc` that executes `srv.GracefulStop()`.

## Configuration Patterns

Servers rely on a specific configuration structure defined in your `config.yaml` (or environment variables).

```yaml
server:
  port: 8080
  tls:
    enabled: false
    cert: "/path/to/cert.pem"
    key: "/path/to/key.pem"
```

The `NewServer` and `Start` functions extract these values automatically from a `config.Containable` object.

## Lifecycle Management with Controls

The primary design goal is to make service registration effortless. By using the provided `Start` and `Stop` curried functions, you can register your servers with a `controls.Controller`.

### Registration Example

```go
// 1. Initialize Server
hServer, _ := httpServer.NewServer(ctx, cfg, mux)

// 2. Register with Controller
ctrl.Register("http_service",
    controls.WithStart(httpServer.Start(cfg, logger, hServer)),
    controls.WithStop(httpServer.Stop(logger, hServer)),
    controls.WithStatus(httpServer.Status()),
)
```

This pattern ensures that when the controller starts or stops, the underlying HTTP or gRPC server follows the same lifecycle, including graceful cleanup of connections.
