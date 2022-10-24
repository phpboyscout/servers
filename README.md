# Servers

This module provides pre-configured HTTP and gRPC server implementations designed to integrate seamlessly with the `github.com/phpboyscout/controls` package for lifecycle management.

## Packages

### `http`

Provides a wrapper around `net/http.Server` with best-practice timeouts and TLS configuration.

### `grpc`

Provides a wrapper around `google.golang.org/grpc.Server` with reflection enabled.

## Usage

Here is an example of how to use these packages with `config` and `controls`:

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"
	"os"

	"github.com/phpboyscout/config"
	"github.com/phpboyscout/controls"
	grpcServer "github.com/phpboyscout/servers/grpc"
	httpServer "github.com/phpboyscout/servers/http"
	"github.com/spf13/afero"
)

func main() {
	ctx := context.Background()
	logger := slog.New(slog.NewTextHandler(os.Stdout, nil))

	// 1. Initialize Configuration
	//    Assumes you have a config.yaml with "server.port" etc.
	cfg, err := config.Load([]string{"config.yaml"}, afero.NewOsFs(), logger, false)
	if err != nil {
		logger.Error(err.Error())
		os.Exit(1)
	}

	// 2. Initialize Controller
	ctrl := controls.NewController(ctx, controls.WithLogger(logger))

	// 3. Configure HTTP Server
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "Hello from HTTP Server")
	})

	hServer, _ := httpServer.NewServer(ctx, cfg, mux)
	
	// Register HTTP server with Controller
	ctrl.Register("http_service",
		controls.WithStart(httpServer.Start(cfg, logger, hServer)),
		controls.WithStop(httpServer.Stop(logger, hServer)),
		controls.WithStatus(httpServer.Status()),
	)

	// 4. Configure gRPC Server
	gServer, _ := grpcServer.NewServer(cfg)
	
	// Register gRPC server with Controller
	ctrl.Register("grpc_service",
		controls.WithStart(grpcServer.Start(cfg, logger, gServer)),
		controls.WithStop(grpcServer.Stop(logger, gServer)),
		controls.WithStatus(grpcServer.Status()),
	)

	// 5. Start all registered services
	ctrl.Start()

	// 6. Wait for shutdown signal (Ctrl+C)
	ctrl.Wait()
}
```

### Configuration Requirements

The servers expect the following configuration structure (example in YAML):

```yaml
server:
  port: 8080
  tls:
    enabled: false
    cert: "/path/to/cert.pem"
    key: "/path/to/key.pem"
```
