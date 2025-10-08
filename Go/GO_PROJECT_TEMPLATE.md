# Go Project Template Guide

This comprehensive guide provides a step-by-step template for initializing new Go projects based on the xbit-agent codebase architecture. It demonstrates best practices for Go project structure, GraphQL APIs, database management, containerization, and microservice patterns.

## Table of Contents

1. [Project Structure & Organization](#1-project-structure--organization)
2. [Initial Setup](#2-initial-setup)
3. [Build System](#3-build-system)
4. [Configuration Management](#4-configuration-management)
5. [Database Management](#5-database-management)
6. [GraphQL Architecture](#6-graphql-architecture)
7. [Background Jobs & Scheduling](#7-background-jobs--scheduling)
8. [Message Queue Integration](#8-message-queue-integration)
9. [Containerization](#9-containerization)
10. [Testing Infrastructure](#10-testing-infrastructure)

## 1. Project Structure & Organization

### Directory Layout

Create the following directory structure for your new Go project:

```
your-project/
├── cmd/                          # Application entry points
│   ├── graphql/                 # GraphQL server
│   ├── scheduler/               # Background job scheduler
│   ├── worker_*/                # Worker services
│   └── cli/                     # CLI tools
├── config/                      # Configuration structures
├── internal/                    # Private application code
│   ├── app/                     # Application initialization
│   ├── controller/              # API controllers and resolvers
│   │   ├── graphql/            # User GraphQL API
│   │   └── admin/              # Admin GraphQL API
│   ├── global/                  # Global variables
│   ├── initialize/              # System initializers
│   ├── initializer/             # Service initializers
│   ├── middleware/              # HTTP/GraphQL middleware
│   ├── model/                   # Database models
│   ├── nats/                    # NATS message queue
│   ├── repo/                    # Repository layer
│   ├── service/                 # Business logic layer
│   ├── task/                    # Background tasks
│   ├── test/                    # Test utilities
│   ├── types/                   # Type definitions
│   └── utils/                   # Utility functions
├── migrations/                  # Database migrations
├── pkg/                         # Public packages
├── scripts/                     # Build and deployment scripts
├── env/                         # Environment files
├── docs/                        # Documentation
├── bin/                         # Compiled binaries
└── sensitive_configs/           # Sensitive configuration files
```

### Package Organization Principles

1. **cmd/**: Each subdirectory represents a different executable
2. **internal/**: Private code that cannot be imported by other projects
3. **pkg/**: Public packages that can be imported by external projects
4. **config/**: Configuration structures and validation
5. **migrations/**: Database schema migrations using Atlas

## 2. Initial Setup

### Step 1: Initialize Go Module

```bash
# Create project directory
mkdir your-project
cd your-project

# Initialize Go module
go mod init github.com/your-org/your-project

# Create basic directory structure
mkdir -p cmd/graphql cmd/scheduler
mkdir -p config internal/{app,controller,global,model,service,repo,utils,test}
mkdir -p migrations scripts env docs bin
```

### Step 2: Essential Dependencies

Create `go.mod` with core dependencies:

```go
module github.com/your-org/your-project

go 1.24.1

require (
    // GraphQL
    github.com/99designs/gqlgen v0.17.78
    github.com/vektah/gqlparser/v2 v2.5.30

    // Web Framework
    github.com/gin-gonic/gin v1.10.1
    github.com/gin-contrib/cors v1.7.6

    // Database
    gorm.io/gorm v1.25.12
    gorm.io/driver/postgres v1.5.11
    ariga.io/atlas-provider-gorm v0.5.3

    // Configuration
    github.com/spf13/viper v1.20.1
    github.com/joho/godotenv v1.5.1

    // Logging
    go.uber.org/zap v1.27.0

    // Message Queue
    github.com/nats-io/nats.go v1.43.0

    // Background Jobs
    github.com/robfig/cron/v3 v3.0.1

    // JWT
    github.com/golang-jwt/jwt/v5 v5.2.3

    // Utilities
    github.com/google/uuid v1.6.0
    github.com/shopspring/decimal v1.4.0

    // Testing
    github.com/stretchr/testify v1.10.0
)
```

### Step 3: Tools Configuration

Create `tools.go` for development tools:

```go
//go:build tools

package tools

import (
    _ "github.com/99designs/gqlgen"
    _ "github.com/99designs/gqlgen/graphql/introspection"
)
```

## 3. Build System

### Makefile Structure

Create a comprehensive `Makefile`:

```makefile
# Project Configuration
Server = .
ServerName = your-project
GOOS = linux
GOARCH = amd64
BinDir = ./bin

# Detect local platform
LOCAL_GOOS = $(shell go env GOOS)
LOCAL_GOARCH = $(shell go env GOARCH)

# Build targets
build: gqlgen-check
	mkdir -p $(BinDir)
	cd $(Server) && GOOS=$(GOOS) GOARCH=$(GOARCH) go build -o $(BinDir)/$(ServerName) cmd/graphql/main.go

build-local: gqlgen-check
	mkdir -p $(BinDir)
	cd $(Server) && GOOS=$(LOCAL_GOOS) GOARCH=$(LOCAL_GOARCH) go build -o $(BinDir)/$(ServerName) cmd/graphql/main.go

# GraphQL generation
gqlgen:
	@echo "Generating GraphQL code..."
	go run github.com/99designs/gqlgen generate --config gqlgen.yml

gqlgen-check:
	@if [ ! -f "internal/controller/graphql/generated.go" ]; then \
		echo "Generated GraphQL files not found, generating..."; \
		$(MAKE) gqlgen; \
	fi

# Database migrations
db-diff:
	atlas migrate diff --env gorm

db-apply:
	./scripts/run.sh local migrate

# Testing
test:
	go test ./...

test-verbose:
	go test -v ./...

test-coverage:
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html

# Development
run-local:
	./scripts/run.sh local run

dev:
	./scripts/run.sh local dev

clean:
	rm -rf $(BinDir)

.PHONY: build build-local gqlgen gqlgen-check db-diff db-apply test test-verbose test-coverage run-local dev clean
```

## 4. Configuration Management

### Environment-Based Configuration

Create `config.yaml` template:

```yaml
# JWT Configuration
jwt:
  signing-key: {{ index . "JWT_SECRET" | default "your-secret-key" }}
  expires-time: {{ index . "JWT_EXPIRES_TIME" | default "7d" }}
  buffer-time: {{ index . "JWT_BUFFER_TIME" | default "1d" }}
  issuer: {{ index . "JWT_ISSUER" | default "your-project" }}

# System Configuration
system:
  env: {{ index . "APP_ENV" | default "local" }}
  addr: {{ index . "SERVER_PORT" | default "8080" }}
  db-type: pgsql
  router-prefix: "/api/your-project"
  graphql-prefix: "/api/your-project/graphql"
  admin-graphql-prefix: "/api/your-project/admin/graphql"

# Database Configuration
pgsql:
  path: {{ index . "POSTGRES_HOST" | default "127.0.0.1" }}
  port: {{ index . "POSTGRES_PORT" | default "5432" }}
  config: sslmode={{ index . "POSTGRES_SSL_MODE" | default "disable" }}
  db-name: {{ index . "POSTGRES_DB" | default "your_project" }}
  username: {{ index . "POSTGRES_USER" | default "postgres" }}
  password: {{ index . "POSTGRES_PASS" | default "postgres" }}
  max-idle-conns: 10
  max-open-conns: 100

# NATS Configuration
nats:
  url: {{ index . "NATS_URL" | default "nats://127.0.0.1:4222" }}
  user: {{ index . "NATS_USER" | default "" }}
  pass: {{ index . "NATS_PASS" | default "" }}
  use-tls: {{ index . "NATS_USE_TLS" | default "false" }}
  token: {{ index . "NATS_TOKEN" | default "" }}

# Scheduled Tasks
cron-tasks:
  - id: "example_task"
    cron: "0 */5 * * * *"  # Every 5 minutes
```

### Environment Files

Create environment files in `env/` directory:

**env/local.env**:
```bash
# Application Configuration
APP_ENV=local
APP_NAME=your-project
SERVER_PORT=8080

# Database Configuration
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASS=postgres
POSTGRES_DB=your_project
POSTGRES_SSL_MODE=disable
DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:5432/your_project?sslmode=disable

# JWT Configuration
JWT_SECRET=local-development-secret-key
JWT_EXPIRES_TIME=7d
JWT_ISSUER=your-project-local

# NATS Configuration
NATS_URL=nats://localhost:4222
```

**env/docker.env**, **env/staging.env**, **env/prod.env** - Similar structure with environment-specific values.

### Configuration Structures

Create `config/config.go`:

```go
package config

type Server struct {
    JWT    JWT    `mapstructure:"jwt" json:"jwt" yaml:"jwt"`
    System System `mapstructure:"system" json:"system" yaml:"system"`
    Pgsql  Pgsql  `mapstructure:"pgsql" json:"pgsql" yaml:"pgsql"`
    Nats   Nats   `mapstructure:"nats" json:"nats" yaml:"nats"`
    CronTasks []Task `mapstructure:"cron-tasks" json:"cron-tasks" yaml:"cron-tasks"`
}

type JWT struct {
    SigningKey  string `mapstructure:"signing-key" json:"signing-key" yaml:"signing-key"`
    ExpiresTime string `mapstructure:"expires-time" json:"expires-time" yaml:"expires-time"`
    BufferTime  string `mapstructure:"buffer-time" json:"buffer-time" yaml:"buffer-time"`
    Issuer      string `mapstructure:"issuer" json:"issuer" yaml:"issuer"`
}

type System struct {
    Env           string `mapstructure:"env" json:"env" yaml:"env"`
    Addr          int    `mapstructure:"addr" json:"addr" yaml:"addr"`
    DbType        string `mapstructure:"db-type" json:"db-type" yaml:"db-type"`
    RouterPrefix  string `mapstructure:"router-prefix" json:"router-prefix" yaml:"router-prefix"`
    GraphqlPrefix string `mapstructure:"graphql-prefix" json:"graphql-prefix" yaml:"graphql-prefix"`
}

type Pgsql struct {
    Path         string `mapstructure:"path" json:"path" yaml:"path"`
    Port         string `mapstructure:"port" json:"port" yaml:"port"`
    Config       string `mapstructure:"config" json:"config" yaml:"config"`
    Dbname       string `mapstructure:"db-name" json:"db-name" yaml:"db-name"`
    Username     string `mapstructure:"username" json:"username" yaml:"username"`
    Password     string `mapstructure:"password" json:"password" yaml:"password"`
    MaxIdleConns int    `mapstructure:"max-idle-conns" json:"max-idle-conns" yaml:"max-idle-conns"`
    MaxOpenConns int    `mapstructure:"max-open-conns" json:"max-open-conns" yaml:"max-open-conns"`
}

type Nats struct {
    URL    string `mapstructure:"url" json:"url" yaml:"url"`
    User   string `mapstructure:"user" json:"user" yaml:"user"`
    Pass   string `mapstructure:"pass" json:"pass" yaml:"pass"`
    UseTLS bool   `mapstructure:"use-tls" json:"use-tls" yaml:"use-tls"`
    Token  string `mapstructure:"token" json:"token" yaml:"token"`
}

type Task struct {
    ID   string `mapstructure:"id" json:"id" yaml:"id"`
    Cron string `mapstructure:"cron" json:"cron" yaml:"cron"`
}
```

### Configuration Loading

Create `internal/initializer/viper.go`:

```go
package initializer

import (
    "bytes"
    "fmt"
    "os"
    "path/filepath"
    "strings"
    "text/template"

    "github.com/fsnotify/fsnotify"
    "github.com/joho/godotenv"
    "github.com/spf13/viper"
    "your-project/internal/global"
)

func Viper(path ...string) *viper.Viper {
    var config string

    if len(path) == 0 {
        config = "config.yaml"
        if configEnv := os.Getenv("GVA_CONFIG"); configEnv != "" {
            config = configEnv
        }
    } else {
        config = path[0]
    }

    // Load environment file based on STAGE
    stage := os.Getenv("STAGE")
    if stage == "" {
        stage = "local"
        os.Setenv("STAGE", stage)
    }

    envFilePath := filepath.Join("env", fmt.Sprintf("%s.env", stage))
    if err := godotenv.Load(envFilePath); err != nil {
        fmt.Printf("Warning: Could not load .env file %s: %v\n", envFilePath, err)
    }

    // Read and parse config template
    configData, err := os.ReadFile(config)
    if err != nil {
        panic(fmt.Errorf("Fatal error reading config file: %s", err))
    }

    // Create template and execute with environment variables
    tmpl, err := template.New("config").Parse(string(configData))
    if err != nil {
        panic(fmt.Errorf("Fatal error parsing config template: %s", err))
    }

    envVars := make(map[string]string)
    for _, env := range os.Environ() {
        pair := strings.SplitN(env, "=", 2)
        if len(pair) == 2 {
            envVars[pair[0]] = pair[1]
        }
    }

    var buf bytes.Buffer
    if err = tmpl.Execute(&buf, envVars); err != nil {
        panic(fmt.Errorf("Fatal error executing config template: %s", err))
    }

    v := viper.New()
    v.SetConfigType("yaml")
    if err = v.ReadConfig(strings.NewReader(buf.String())); err != nil {
        panic(fmt.Errorf("Fatal error reading config: %s", err))
    }

    v.WatchConfig()
    v.OnConfigChange(func(e fsnotify.Event) {
        fmt.Println("Config file changed:", e.Name)
        if err = v.Unmarshal(&global.GVA_CONFIG); err != nil {
            fmt.Println(err)
        }
    })

    if err = v.Unmarshal(&global.GVA_CONFIG); err != nil {
        panic(err)
    }

    return v
}
```

## 5. Database Management

### Atlas Migration Setup

Create `atlas.hcl`:

```hcl
data "external_schema" "gorm" {
  program = [
    "go",
    "run",
    "./cmd/atlasloader",
  ]
}

env "gorm" {
  src = data.external_schema.gorm.url
  dev = "docker://postgres/14.1/dev"
  migration {
    dir = "file://migrations"
  }
  format {
    migrate {
      diff = "{{ sql . \"  \" }}"
    }
  }
}
```

### Atlas Loader

Create `cmd/atlasloader/main.go`:

```go
package main

import (
    "fmt"
    "io"
    "os"

    "ariga.io/atlas-provider-gorm/gormschema"
    "your-project/internal/model"
)

func main() {
    stmts, err := gormschema.New("postgres").Load(
        &model.User{},
        &model.Product{},
        // Add your models here
    )
    if err != nil {
        fmt.Fprintf(os.Stderr, "failed to load gorm schema: %v\n", err)
        os.Exit(1)
    }
    io.WriteString(os.Stdout, stmts)
}
```

### Database Models

Create `internal/model/user.go`:

```go
package model

import (
    "time"
    "github.com/google/uuid"
    "gorm.io/gorm"
)

type User struct {
    ID        uuid.UUID      `gorm:"type:uuid;primary_key;default:gen_random_uuid()" json:"id"`
    Email     *string        `gorm:"type:varchar(255);uniqueIndex" json:"email"`
    Name      string         `gorm:"type:varchar(255);not null" json:"name"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"deleted_at"`
}

func (User) TableName() string {
    return "users"
}
```

### Database Initialization

Create `internal/initializer/gorm.go`:

```go
package initializer

import (
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
    "your-project/internal/global"
)

func Gorm() *gorm.DB {
    switch global.GVA_CONFIG.System.DbType {
    case "postgres":
        return GormPostgres()
    default:
        return GormPostgres()
    }
}

func GormPostgres() *gorm.DB {
    p := global.GVA_CONFIG.Pgsql
    if p.Dbname == "" {
        return nil
    }

    dsn := fmt.Sprintf("host=%s user=%s password=%s dbname=%s port=%s %s",
        p.Path, p.Username, p.Password, p.Dbname, p.Port, p.Config)

    postgresConfig := postgres.Config{
        DSN:                  dsn,
        PreferSimpleProtocol: false,
    }

    if db, err := gorm.Open(postgres.New(postgresConfig), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Silent),
    }); err != nil {
        panic("Failed to connect to database: " + err.Error())
    } else {
        sqlDB, _ := db.DB()
        sqlDB.SetMaxIdleConns(p.MaxIdleConns)
        sqlDB.SetMaxOpenConns(p.MaxOpenConns)
        return db
    }
}
```

### Migration Workflow

1. **Create/Modify Models**: Add or update your GORM models
2. **Generate Migration**: `make db-diff`
3. **Apply Migration**: `make db-apply`

## 6. GraphQL Architecture

### GraphQL Configuration

Create `gqlgen.yml`:

```yaml
schema:
  - internal/controller/graphql/*.graphqls
  - internal/controller/graphql/schemas/*.gql

exec:
  filename: internal/controller/graphql/generated.go
  package: graphql

model:
  filename: internal/controller/graphql/gql_model/models_gen.go
  package: gql_model

resolver:
  layout: follow-schema
  dir: internal/controller/graphql/
  package: graphql
  filename_template: "{name}.resolvers.go"

directives:
  auth:
    implementation: AuthDirective

models:
  ID:
    model:
      - github.com/99designs/gqlgen/graphql.ID
  Time:
    model:
      - github.com/99designs/gqlgen/graphql.Time
```

### GraphQL Schema

Create `internal/controller/graphql/schema.graphqls`:

```graphql
directive @auth on FIELD_DEFINITION

type Query {
  users: [User!]! @auth
  user(id: ID!): User @auth
}

type Mutation {
  createUser(input: CreateUserInput!): User! @auth
  updateUser(id: ID!, input: UpdateUserInput!): User! @auth
}

type User {
  id: ID!
  email: String
  name: String!
  createdAt: Time!
  updatedAt: Time!
}

input CreateUserInput {
  email: String
  name: String!
}

input UpdateUserInput {
  email: String
  name: String
}

scalar Time
```

### GraphQL Resolvers

Create `internal/controller/graphql/resolver.go`:

```go
package graphql

import (
    "your-project/internal/service"
)

type Resolver struct {
    userService service.UserServiceInterface
}

func NewRootResolver() *Resolver {
    return &Resolver{
        userService: service.NewUserService(),
    }
}
```

### Authentication Directive

Create `internal/controller/graphql/directive.go`:

```go
package graphql

import (
    "context"
    "fmt"

    "github.com/99designs/gqlgen/graphql"
    "github.com/google/uuid"
)

func AuthDirective(ctx context.Context, obj interface{}, next graphql.Resolver) (interface{}, error) {
    userID := ctx.Value("userId")
    if userID == nil {
        return nil, fmt.Errorf("access denied: authentication required")
    }

    if _, err := uuid.Parse(userID.(string)); err != nil {
        return nil, fmt.Errorf("access denied: invalid user ID")
    }

    return next(ctx)
}
```

## 7. Background Jobs & Scheduling

### Task Scheduler

Create `internal/task/task.go`:

```go
package task

import (
    "context"
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"

    "github.com/robfig/cron/v3"
)

type TaskScheduler struct {
    cron       *cron.Cron
    tasks      map[string]cron.EntryID
    cancelFunc context.CancelFunc
}

func NewTaskScheduler() *TaskScheduler {
    return &TaskScheduler{
        cron:  cron.New(cron.WithSeconds()),
        tasks: make(map[string]cron.EntryID),
    }
}

func (s *TaskScheduler) Register(id string, expr string, job func()) error {
    if _, exists := s.tasks[id]; exists {
        return fmt.Errorf("task ID %s already exists", id)
    }

    entryID, err := s.cron.AddFunc(expr, job)
    if err != nil {
        return fmt.Errorf("register task failed: %w", err)
    }

    s.tasks[id] = entryID
    log.Printf("✅ Registered task [%s] with cron: %s\n", id, expr)
    return nil
}

func (s *TaskScheduler) Start() {
    s.cron.Start()
    log.Println("🚀 Task scheduler started")
}

func (s *TaskScheduler) Stop() {
    ctx := s.cron.Stop()
    <-ctx.Done()
    log.Println("🛑 Task scheduler stopped")
}

func (s *TaskScheduler) RunWithSignal(ctx context.Context) {
    go s.Start()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

    select {
    case <-quit:
        log.Println("🔔 Received exit signal, stopping scheduler...")
        s.Stop()
        if s.cancelFunc != nil {
            s.cancelFunc()
        }
    case <-ctx.Done():
        log.Println("🛑 Context cancelled, stopping scheduler...")
        s.Stop()
    }
}
```

### Task Registration

Create `internal/initializer/task.go`:

```go
package initializer

import (
    "context"
    "your-project/internal/global"
    "your-project/internal/task"
    "your-project/internal/utils"
    "go.uber.org/zap"
)

func InitTask() {
    scheduler := task.NewTaskScheduler()
    ctx, cancel := context.WithCancel(context.Background())
    scheduler.SetCancelFunc(cancel)
    go scheduler.RunWithSignal(ctx)

    // Register your tasks here
    for _, taskConfig := range global.GVA_CONFIG.CronTasks {
        switch taskConfig.ID {
        case "example_task":
            err := scheduler.Register(taskConfig.ID, taskConfig.Cron, func() {
                // Your task logic here
                global.GVA_LOG.Info("Executing example task")
            })
            if err != nil {
                global.GVA_LOG.Error("Failed to register example task", zap.Error(err))
            }
        }
    }
}
```

### Scheduler Service

Create `cmd/scheduler/main.go`:

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"

    "your-project/internal/global"
    "your-project/internal/initializer"
    "go.uber.org/zap"
)

func main() {
    global.GVA_VP = initializer.Viper()
    global.GVA_LOG = initializer.Zap()
    zap.ReplaceGlobals(global.GVA_LOG)
    global.GVA_DB = initializer.Gorm()

    // Initialize scheduled tasks
    go initializer.InitTask()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    fmt.Println("Shutting down scheduler...")
}
```

## 8. Message Queue Integration

### NATS Client Setup

Create `internal/nats/nats.go`:

```go
package nats

import (
    "crypto/tls"
    "log"
    "time"

    "github.com/nats-io/nats.go"
    "your-project/config"
)

type NATSClient struct {
    conn *nats.Conn
    js   nats.JetStreamContext
}

func InitNatsJetStream(natConfig config.Nats) *NATSClient {
    var tlsConfig *tls.Config
    if natConfig.UseTLS {
        tlsConfig = &tls.Config{
            MinVersion: tls.VersionTLS12,
        }
    }

    natsOptions := []nats.Option{
        nats.Timeout(10 * time.Second),
        nats.MaxReconnects(-1),
        nats.ReconnectWait(2 * time.Second),
        nats.DisconnectErrHandler(func(nc *nats.Conn, err error) {
            log.Printf("NATS disconnected: %v", err)
        }),
        nats.ReconnectHandler(func(nc *nats.Conn) {
            log.Printf("NATS reconnected to %s", nc.ConnectedUrl())
        }),
    }

    if tlsConfig != nil {
        natsOptions = append(natsOptions, nats.Secure(tlsConfig))
    }

    if natConfig.Token != "" {
        natsOptions = append(natsOptions, nats.Token(natConfig.Token))
    } else {
        natsOptions = append(natsOptions, nats.UserInfo(natConfig.User, natConfig.Pass))
    }

    nc, err := nats.Connect(natConfig.URL, natsOptions...)
    if err != nil {
        log.Fatalf("Cannot connect to NATS: %v", err)
    }

    js, err := nc.JetStream()
    if err != nil {
        log.Fatalf("Cannot create JetStream: %v", err)
    }

    log.Println("Connected to NATS JetStream successfully!")
    return &NATSClient{conn: nc, js: js}
}

func (n *NATSClient) Publish(subject string, data []byte) error {
    return n.conn.Publish(subject, data)
}

func (n *NATSClient) PublishJS(subject string, data []byte, opts ...nats.PubOpt) (*nats.PubAck, error) {
    return n.js.Publish(subject, data, opts...)
}

func (n *NATSClient) Subscribe(subject string, handler nats.MsgHandler) (*nats.Subscription, error) {
    return n.conn.Subscribe(subject, handler)
}

func (n *NATSClient) SubscribeJS(subject string, handler nats.MsgHandler, opts ...nats.SubOpt) (*nats.Subscription, error) {
    return n.js.Subscribe(subject, handler, opts...)
}

func (n *NATSClient) Close() {
    if n.conn != nil && n.conn.Status() == nats.CONNECTED {
        log.Println("Closing NATS connection...")
        n.conn.Close()
    }
}
```

### Publisher/Subscriber Patterns

Create `internal/nats/publisher.go`:

```go
package nats

import "github.com/nats-io/nats.go"

type Publisher interface {
    Publish(subject string, data []byte) error
    PublishJS(subject string, data []byte, opts ...nats.PubOpt) (*nats.PubAck, error)
}
```

Create `internal/nats/subscriber.go`:

```go
package nats

import "github.com/nats-io/nats.go"

type Subscriber interface {
    Subscribe(subject string, handler nats.MsgHandler) (*nats.Subscription, error)
    SubscribeJS(subject string, handler nats.MsgHandler, opts ...nats.SubOpt) (*nats.Subscription, error)
}
```

### Message Models

Create `internal/nats/model.go`:

```go
package nats

import "time"

// Stream and Subject constants
const (
    // Example stream
    ExampleStream = "example-stream"

    // Example subjects
    ExampleSubject = "example.event"

    // Consumer names
    ExampleConsumer = "example-consumer"
)

// Example message structure
type ExampleMessage struct {
    ID        string                 `json:"id"`
    Type      string                 `json:"type"`
    Data      map[string]interface{} `json:"data"`
    Timestamp time.Time              `json:"timestamp"`
}
```

### Worker Service Example

Create `cmd/worker_example/main.go`:

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/nats-io/nats.go"
    "your-project/internal/global"
    "your-project/internal/initializer"
    natsClient "your-project/internal/nats"
)

func main() {
    // Initialize configuration and services
    global.GVA_VP = initializer.Viper()
    global.GVA_LOG = initializer.Zap()
    global.GVA_DB = initializer.Gorm()

    // Initialize NATS client
    nc := natsClient.InitNatsJetStream(global.GVA_CONFIG.Nats)
    defer nc.Close()

    // Start worker
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    go runWorker(ctx, nc)

    // Wait for shutdown signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down worker...")
    cancel()
}

func runWorker(ctx context.Context, nc *natsClient.NATSClient) {
    // Subscribe to messages
    _, err := nc.SubscribeJS(natsClient.ExampleSubject, func(msg *nats.Msg) {
        var exampleMsg natsClient.ExampleMessage
        if err := json.Unmarshal(msg.Data, &exampleMsg); err != nil {
            log.Printf("Failed to unmarshal message: %v", err)
            msg.Nak()
            return
        }

        // Process message
        log.Printf("Processing message: %+v", exampleMsg)

        // Simulate processing time
        time.Sleep(100 * time.Millisecond)

        // Acknowledge message
        msg.Ack()
    }, nats.Durable(natsClient.ExampleConsumer), nats.ManualAck())

    if err != nil {
        log.Fatalf("Failed to subscribe: %v", err)
    }

    <-ctx.Done()
}
```

## 9. Containerization

### Dockerfile

Create `Dockerfile`:

```dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /app

# Install dependencies
RUN apk add --no-cache git make

# Copy go mod and sum files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Copy source code
COPY . .

# Generate GraphQL code before building
RUN make gqlgen

# Build the applications
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main/graphql cmd/graphql/main.go
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main/scheduler cmd/scheduler/main.go
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main/worker_example cmd/worker_example/main.go

# Final stage
FROM alpine:latest

# Install ca-certificates and atlas
RUN apk --no-cache add ca-certificates curl
RUN curl -sSf https://atlasgo.sh | sh

WORKDIR /root/

# Copy binaries and configuration
COPY --from=builder /app/main/ .
COPY --from=builder /app/config.yaml .
COPY --from=builder /app/migrations ./migrations
COPY --from=builder /app/entrypoint.sh .
COPY env/ env/

# Make entrypoint executable
RUN chmod +x entrypoint.sh

# Expose port
EXPOSE 8080

# Set entrypoint
ENTRYPOINT ["./entrypoint.sh"]

# Default command
CMD ["./graphql"]
```

### Entrypoint Script

Create `entrypoint.sh`:

```bash
#!/bin/sh
set -e

_terminate() {
  echo "Signal received. Waiting for graceful shutdown..."
  kill $(jobs -p)
  wait
  exit 0
}

trap _terminate SIGINT SIGTERM

# Run database migrations
export DATABASE_URL="postgresql://$POSTGRES_USER:$POSTGRES_PASS@$POSTGRES_HOST:$POSTGRES_PORT/$POSTGRES_DB?sslmode=${POSTGRES_SSL_MODE:-disable}"
atlas migrate apply --url $DATABASE_URL --dir file://migrations

# Execute the main command
exec $@
```

### Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - STAGE=docker
    depends_on:
      - postgres
      - nats
    volumes:
      - ./env:/root/env

  scheduler:
    build: .
    command: ["./scheduler"]
    environment:
      - STAGE=docker
    depends_on:
      - postgres
      - nats
    volumes:
      - ./env:/root/env

  postgres:
    image: postgres:14
    environment:
      POSTGRES_DB: your_project
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  nats:
    image: nats:2.9-alpine
    ports:
      - "4222:4222"
      - "8222:8222"
    command: ["--jetstream", "--store_dir", "/data"]
    volumes:
      - nats_data:/data

volumes:
  postgres_data:
  nats_data:
```

## 10. Testing Infrastructure

### Test Configuration

Create `internal/test/config.go`:

```go
package test

import (
    "path/filepath"
    "runtime"

    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
    "your-project/config"
    "your-project/internal/global"
)

type TestConfig struct {
    Server *config.Server
}

func SetupTestConfig() *TestConfig {
    _, filename, _, _ := runtime.Caller(0)
    projectRoot := filepath.Dir(filepath.Dir(filepath.Dir(filename)))

    testConfig := &TestConfig{
        Server: &config.Server{
            JWT: config.JWT{
                SigningKey:  "test-secret-key",
                ExpiresTime: "24h",
                BufferTime:  "1h",
                Issuer:      "your-project-test",
            },
            System: config.System{
                Env:    "test",
                Addr:   8080,
                DbType: "sqlite",
            },
        },
    }

    global.GVA_CONFIG = *testConfig.Server
    return testConfig
}

func SetupTestDB() *gorm.DB {
    db, err := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Silent),
    })
    if err != nil {
        panic("Failed to connect to test database: " + err.Error())
    }

    global.GVA_DB = db
    return db
}

func CleanupTestDB() {
    if global.GVA_DB != nil {
        sqlDB, err := global.GVA_DB.DB()
        if err == nil {
            sqlDB.Close()
        }
        global.GVA_DB = nil
    }
}
```

### Test Fixtures

Create `internal/test/fixtures.go`:

```go
package test

import (
    "time"
    "github.com/google/uuid"
    "your-project/internal/model"
)

type TestFixtures struct{}

func NewTestFixtures() *TestFixtures {
    return &TestFixtures{}
}

func (f *TestFixtures) CreateTestUser() *model.User {
    userID := uuid.New()
    email := "test@example.com"

    return &model.User{
        ID:        userID,
        Email:     &email,
        Name:      "Test User",
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }
}

func (f *TestFixtures) CreateTestUserWithEmail(email string) *model.User {
    user := f.CreateTestUser()
    user.Email = &email
    return user
}
```

### Test Helpers

Create `internal/test/helpers.go`:

```go
package test

import (
    "context"
    "testing"

    "github.com/google/uuid"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

type TestHelper struct {
    t *testing.T
}

func NewTestHelper(t *testing.T) *TestHelper {
    return &TestHelper{t: t}
}

func (h *TestHelper) AssertNoError(err error) {
    assert.NoError(h.t, err)
}

func (h *TestHelper) AssertError(err error) {
    assert.Error(h.t, err)
}

func (h *TestHelper) AssertEqual(expected, actual interface{}) {
    assert.Equal(h.t, expected, actual)
}

func (h *TestHelper) AssertNotNil(object interface{}) {
    assert.NotNil(h.t, object)
}

func (h *TestHelper) AssertNil(object interface{}) {
    assert.Nil(h.t, object)
}

func (h *TestHelper) RequireNoError(err error) {
    require.NoError(h.t, err)
}

func (h *TestHelper) CreateTestContext() context.Context {
    return context.Background()
}

func (h *TestHelper) CreateTestContextWithUserID(userID uuid.UUID) context.Context {
    ctx := context.Background()
    return context.WithValue(ctx, "userId", userID.String())
}

func (h *TestHelper) GenerateTestUUID() uuid.UUID {
    return uuid.New()
}

// Setup functions
func SetupTest(t *testing.T) (*TestConfig, *TestFixtures, *TestHelper) {
    config := SetupTestConfig()
    fixtures := NewTestFixtures()
    helper := NewTestHelper(t)
    return config, fixtures, helper
}

func SetupTestWithDB(t *testing.T) (*TestConfig, *TestFixtures, *TestHelper) {
    config := SetupTestConfig()
    SetupTestDB()
    fixtures := NewTestFixtures()
    helper := NewTestHelper(t)
    return config, fixtures, helper
}

func TeardownTest() {
    CleanupTestDB()
}
```

### Example Test

Create `internal/service/user_service_test.go`:

```go
package service

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "your-project/internal/test"
)

func TestUserService_CreateUser(t *testing.T) {
    config, fixtures, helper := test.SetupTestWithDB(t)
    defer test.TeardownTest()

    ctx := helper.CreateTestContext()
    user := fixtures.CreateTestUser()

    // Test user creation logic here
    assert.NotNil(t, user)
    assert.NotEmpty(t, user.ID)
}

func TestUserService_GetUser(t *testing.T) {
    config, fixtures, helper := test.SetupTestWithDB(t)
    defer test.TeardownTest()

    ctx := helper.CreateTestContext()
    user := fixtures.CreateTestUser()

    // Test user retrieval logic here
    helper.AssertNotNil(user)
    helper.AssertEqual("Test User", user.Name)
}
```

## 11. Scripts and Automation

### Run Script

Create `scripts/run.sh`:

```bash
#!/bin/bash
set -e

# Load environment variables
source scripts/load-env.sh $1

case "$2" in
    "migrate")
        echo "Running database migrations..."
        atlas migrate apply --url "$DATABASE_URL" --dir file://migrations
        ;;
    "run")
        echo "Starting application..."
        go run cmd/graphql/main.go
        ;;
    "scheduler")
        echo "Starting scheduler..."
        go run cmd/scheduler/main.go
        ;;
    "dev")
        echo "Starting development server with hot reload..."
        # Install air if not present
        if ! command -v air &> /dev/null; then
            go install github.com/cosmtrek/air@latest
        fi
        air -c .air.toml
        ;;
    *)
        echo "Usage: $0 <env> <command>"
        echo "Commands: migrate, run, scheduler, dev"
        exit 1
        ;;
esac
```

### Environment Loading Script

Create `scripts/load-env.sh`:

```bash
#!/bin/bash
set -e

ENV=${1:-local}
ENV_FILE="env/${ENV}.env"

if [ ! -f "$ENV_FILE" ]; then
    echo "Error: Environment file '$ENV_FILE' not found!"
    exit 1
fi

echo "Loading environment variables from: $ENV_FILE"
export $(grep -v '^#' "$ENV_FILE" | grep -v '^$' | xargs)

# Verify critical variables
REQUIRED_VARS=(
    "POSTGRES_HOST"
    "POSTGRES_PORT"
    "POSTGRES_USER"
    "POSTGRES_PASS"
    "POSTGRES_DB"
    "SERVER_PORT"
)

for var in "${REQUIRED_VARS[@]}"; do
    if [ -z "${!var}" ]; then
        echo "Error: Required environment variable '$var' is not set!"
        exit 1
    fi
done

echo "Environment variables loaded successfully!"
export XBIT_AGENT_ENV="$ENV"
```

## 12. Project Initialization Checklist

### Step-by-Step Setup

1. **Create Project Structure**
   ```bash
   mkdir your-project && cd your-project
   go mod init github.com/your-org/your-project
   # Create directory structure as shown above
   ```

2. **Copy Template Files**
   - Copy all configuration files (`config.yaml`, `atlas.hcl`, etc.)
   - Copy environment files (`env/*.env`)
   - Copy scripts (`scripts/*.sh`)
   - Copy Dockerfile and docker-compose.yml

3. **Update Module References**
   - Replace `your-project` with your actual module name in all Go files
   - Update import paths throughout the codebase

4. **Install Dependencies**
   ```bash
   go mod tidy
   make install-deps
   ```

5. **Setup Database**
   ```bash
   # Start PostgreSQL (via Docker or local installation)
   make db-apply  # Apply initial migrations
   ```

6. **Generate GraphQL Code**
   ```bash
   make gqlgen
   ```

7. **Build and Test**
   ```bash
   make build-local
   make test
   ```

8. **Run Development Server**
   ```bash
   make dev
   ```

### Customization Points

1. **Business Logic**: Replace example models and services with your domain logic
2. **GraphQL Schema**: Update schema files to match your API requirements
3. **Database Models**: Add your specific GORM models
4. **Background Tasks**: Implement your scheduled jobs in the task system
5. **Message Queues**: Configure NATS subjects and consumers for your use case
6. **Authentication**: Customize JWT handling and user authentication
7. **Environment Configuration**: Update environment variables for your deployment

### Best Practices

1. **Follow Go Conventions**: Use standard Go project layout and naming conventions
2. **Separate Concerns**: Keep business logic in services, data access in repositories
3. **Test Coverage**: Write comprehensive unit and integration tests
4. **Configuration Management**: Use environment-specific configurations
5. **Error Handling**: Implement consistent error handling patterns
6. **Logging**: Use structured logging with appropriate log levels
7. **Documentation**: Maintain clear API documentation and code comments
8. **Security**: Implement proper authentication, authorization, and input validation

This template provides a solid foundation for building scalable Go applications with GraphQL APIs, background job processing, and microservice architecture patterns. Customize it according to your specific business requirements while maintaining the architectural principles demonstrated in the xbit-agent reference implementation.
```
