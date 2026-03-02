# Solid Queue Autoscaler

[![CI](https://github.com/reillyse/solid_queue_autoscaler/actions/workflows/ci.yml/badge.svg)](https://github.com/reillyse/solid_queue_autoscaler/actions/workflows/ci.yml)
[![Gem Version](https://badge.fury.io/rb/solid_queue_autoscaler.svg)](https://badge.fury.io/rb/solid_queue_autoscaler)

A control plane for [Solid Queue](https://github.com/rails/solid_queue) that automatically scales worker processes based on queue metrics. Supports both **Heroku** and **Kubernetes** deployments.

## Features

- **Metrics-based scaling**: Scales based on queue depth, job latency, and throughput
- **Multiple scaling strategies**: Fixed increment or proportional scaling based on load
- **Multi-worker support**: Configure and scale different worker types independently
- **Scale to zero**: Full support for `min_workers = 0` to eliminate costs during idle periods
- **Platform adapters**: Native support for Heroku and Kubernetes
- **Singleton execution**: Uses PostgreSQL advisory locks to ensure only one autoscaler runs at a time
- **Safety features**: Cooldowns, min/max limits, dry-run mode
- **Rails integration**: Configuration via initializer, Railtie with rake tasks
- **Flexible execution**: Run as a recurring Solid Queue job or standalone

## Scale to Zero

The autoscaler fully supports scaling workers to zero (`min_workers = 0`), allowing you to eliminate worker costs during idle periods.

### How It Works

When you configure `min_workers = 0` and the queue becomes idle, the autoscaler will scale your workers down to zero. This is ideal for:

- **Development/staging environments** with sporadic usage
- **Batch processing workers** that only run when jobs are queued
- **Cost-sensitive applications** with predictable idle periods

### Heroku Formation Behavior

On Heroku, when a dyno type is scaled to 0, it gets **removed from the formation entirely**. This means:

1. `heroku ps:scale worker=0` removes the `worker` formation
2. Subsequent API calls to get formation info return **404 Not Found**
3. When scaling back up, the formation must be **recreated**

As of **v1.0.15**, the autoscaler handles this gracefully:

- When querying a non-existent formation, it returns `0` workers (instead of raising an error)
- When scaling up a non-existent formation, it automatically creates it using Heroku's batch update API
- This enables seamless scale-to-zero → scale-up workflows

### Configuration Example

```ruby
SolidQueueAutoscaler.configure(:batch_worker) do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'batch_worker'

  # Enable scale-to-zero
  config.min_workers = 0
  config.max_workers = 5

  # Scale up immediately when any job is queued
  config.scale_up_queue_depth = 1
  config.scale_up_latency_seconds = 60

  # Scale down when completely idle
  config.scale_down_queue_depth = 0
  config.scale_down_latency_seconds = 10

  # Longer scale-down cooldown to avoid premature scaling to zero
  config.scale_up_cooldown_seconds = 30
  config.scale_down_cooldown_seconds = 300  # 5 minutes
end
```

### Important Considerations

**Cold-start latency**: When workers are at zero and a job is enqueued, there will be latency before the job is processed:
1. The autoscaler job must run (depends on your `schedule` interval)
2. The autoscaler must scale up workers
3. Heroku must provision and start the dyno (~10-30 seconds)
4. The worker must boot and start processing

Total cold-start time is typically **30-90 seconds** depending on your configuration and dyno startup time.

### Faster Scale-from-Zero (v1.0.20+)

As of **v1.0.20**, the autoscaler includes optimizations for faster cold starts when scaling from zero:

1. **Lower thresholds at zero**: When workers are at 0 (with `min_workers = 0`), the autoscaler uses separate, more aggressive thresholds:
   - `scale_from_zero_queue_depth` (default: 1) - Scale up when there's at least 1 job
   - `scale_from_zero_latency_seconds` (default: 1.0) - Job must be at least 1 second old

2. **Cooldown bypass**: Cooldowns are skipped when scaling from 0 workers, ensuring the fastest possible response.

3. **Grace period for other workers**: The `scale_from_zero_latency_seconds` setting (default: 1 second) ensures that if you have multiple worker types, other workers have a brief chance to pick up the job before a new dyno is spun up.

**Example configuration:**

```ruby
SolidQueueAutoscaler.configure(:batch_worker) do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'batch_worker'

  # Enable scale-to-zero
  config.min_workers = 0
  config.max_workers = 5

  # Normal scaling thresholds (used when workers > 0)
  config.scale_up_queue_depth = 100
  config.scale_up_latency_seconds = 300

  # Scale-from-zero thresholds (used when workers == 0)
  config.scale_from_zero_queue_depth = 1        # Scale up with just 1 job
  config.scale_from_zero_latency_seconds = 2.0  # Wait 2 seconds for other workers
end
```

**Where to run the autoscaler**: The autoscaler job **must run on a process that's always running** (like your web dyno), NOT on the workers being scaled. If the autoscaler runs on workers and those workers scale to zero, there's nothing to scale them back up!

```yaml
# config/recurring.yml - runs on whatever process runs the dispatcher
autoscaler_batch:
  class: SolidQueueAutoscaler::AutoscaleJob
  queue: autoscaler
  schedule: every 30 seconds
  args: [:batch_worker]
```

**Procfile setup**: Ensure your web dyno runs the Solid Queue dispatcher (or use a dedicated always-on dyno):

```
# Procfile
web: bundle exec puma -C config/puma.rb
worker: bundle exec rake solid_queue:start
batch_worker: bundle exec rake solid_queue:start
```

Alternatively, run the dispatcher in a thread within your web process using `solid_queue.yml` configuration.

## Installation

Add to your Gemfile:

```ruby
gem 'solid_queue_autoscaler'
```

Then run:

```bash
bundle install
```

### Database Setup (Recommended)

For persistent cooldown tracking that survives process restarts:

```bash
rails generate solid_queue_autoscaler:migration
rails db:migrate
```

This creates a `solid_queue_autoscaler_state` table to store cooldown timestamps.

### Dashboard Setup (Optional)

For a web UI to monitor autoscaler events and status:

```bash
rails generate solid_queue_autoscaler:dashboard
rails db:migrate
```

Then mount the dashboard in `config/routes.rb`:

```ruby
# With authentication (recommended)
authenticate :user, ->(u) { u.admin? } do
  mount SolidQueueAutoscaler::Dashboard::Engine => "/autoscaler"
end

# Or without authentication
mount SolidQueueAutoscaler::Dashboard::Engine => "/autoscaler"
```

## Quick Start

### Basic Configuration (Single Worker)

Create an initializer at `config/initializers/solid_queue_autoscaler.rb`:

```ruby
SolidQueueAutoscaler.configure do |config|
  # Platform: Heroku
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'worker'

  # Worker limits
  config.min_workers = 1
  config.max_workers = 10

  # Scaling thresholds
  config.scale_up_queue_depth = 100
  config.scale_up_latency_seconds = 300
  config.scale_down_queue_depth = 10
  config.scale_down_latency_seconds = 30
end
```

### Multi-Worker Configuration

Scale different worker types independently with named configurations:

```ruby
# Critical jobs worker - fast response, dedicated queue
SolidQueueAutoscaler.configure(:critical_worker) do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'critical_worker'
  
  # Only monitor the critical queue
  config.queues = ['critical']
  
  # Aggressive scaling for critical jobs
  config.min_workers = 2
  config.max_workers = 20
  config.scale_up_queue_depth = 10
  config.scale_up_latency_seconds = 30
  config.cooldown_seconds = 60
end

# Default worker - handles standard queues
SolidQueueAutoscaler.configure(:default_worker) do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'worker'
  
  # Monitor default and mailers queues
  config.queues = ['default', 'mailers']
  
  # Conservative scaling for background jobs
  config.min_workers = 1
  config.max_workers = 10
  config.scale_up_queue_depth = 100
  config.scale_up_latency_seconds = 300
  config.cooldown_seconds = 120
end

# Batch processing worker - handles long-running jobs
SolidQueueAutoscaler.configure(:batch_worker) do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'batch_worker'
  
  config.queues = ['batch', 'imports', 'exports']
  
  config.min_workers = 0
  config.max_workers = 5
  config.scale_up_queue_depth = 1  # Scale up when any batch job is queued
  config.scale_down_queue_depth = 0
  config.cooldown_seconds = 300
end
```

## Platform Adapters

### Heroku Adapter (Default)

```ruby
SolidQueueAutoscaler.configure do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'worker'  # Dyno type to scale
end
```

Generate a Heroku API key:

```bash
heroku authorizations:create -d "Solid Queue Autoscaler"
```

### Kubernetes Adapter

```ruby
SolidQueueAutoscaler.configure do |config|
  config.adapter = :kubernetes
  config.kubernetes_namespace = ENV.fetch('KUBERNETES_NAMESPACE', 'default')
  config.kubernetes_deployment = 'solid-queue-worker'
  
  # Optional: Custom kubeconfig path (defaults to in-cluster config)
  # config.kubernetes_config_path = '/path/to/kubeconfig'
end
```

The Kubernetes adapter uses the official `kubeclient` gem and supports:
- In-cluster service account authentication (recommended for production)
- External kubeconfig file authentication (useful for development)

## Configuration Reference

### Core Settings

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `adapter` | Symbol | `:heroku` | Platform adapter (`:heroku` or `:kubernetes`) |
| `enabled` | Boolean | `true` | Master switch to enable/disable autoscaling |
| `dry_run` | Boolean | `false` | Log decisions without making changes |
| `queues` | Array | `nil` | Queue names to monitor (nil = all queues) |
| `table_prefix` | String | `'solid_queue_'` | Solid Queue table name prefix |
| `database_connection` | Connection | auto | Database connection (auto-detects `SolidQueue::Record.connection`) |

### Worker Limits

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `min_workers` | Integer | `1` | Minimum workers to maintain |
| `max_workers` | Integer | `10` | Maximum workers allowed |

### Scale-Up Thresholds

Scaling up triggers when **ANY** threshold is exceeded:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `scale_up_queue_depth` | Integer | `100` | Jobs in queue to trigger scale up |
| `scale_up_latency_seconds` | Integer | `300` | Oldest job age to trigger scale up |
| `scale_up_increment` | Integer | `1` | Workers to add (fixed strategy) |

### Scale-Down Thresholds

Scaling down triggers when **ALL** thresholds are met:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `scale_down_queue_depth` | Integer | `10` | Jobs in queue threshold |
| `scale_down_latency_seconds` | Integer | `30` | Oldest job age threshold |
| `scale_down_decrement` | Integer | `1` | Workers to remove |

### Scaling Strategies

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `scaling_strategy` | Symbol | `:fixed` | `:fixed` or `:proportional` |
| `scale_up_jobs_per_worker` | Integer | `50` | Jobs per worker (proportional) |
| `scale_up_latency_per_worker` | Integer | `60` | Seconds per worker (proportional) |
| `scale_down_jobs_per_worker` | Integer | `50` | Jobs capacity per worker |

### Cooldowns

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `cooldown_seconds` | Integer | `120` | Default cooldown for both directions |
| `scale_up_cooldown_seconds` | Integer | `nil` | Override for scale-up cooldown |
| `scale_down_cooldown_seconds` | Integer | `nil` | Override for scale-down cooldown |
| `persist_cooldowns` | Boolean | `true` | Save cooldowns to database |

### Scale-from-Zero Optimization

These settings control the faster cold-start behavior when `min_workers = 0` and workers are currently at 0:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `scale_from_zero_queue_depth` | Integer | `1` | Jobs in queue to trigger scale-up when at 0 workers |
| `scale_from_zero_latency_seconds` | Float | `1.0` | Job must be at least this old (gives other workers a chance) |

**Note:** When scaling from 0 workers, cooldowns are automatically bypassed for the fastest possible response.

### AutoscaleJob Settings

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `job_queue` | Symbol/String | `:autoscaler` | Queue for the AutoscaleJob |
| `job_priority` | Integer | `nil` | Priority for the AutoscaleJob (lower = higher priority) |

### Heroku-Specific

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `heroku_api_key` | String | `nil` | Heroku Platform API token |
| `heroku_app_name` | String | `nil` | Heroku app name |
| `process_type` | String | `'worker'` | Dyno type to scale |

### Kubernetes-Specific

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `kubernetes_namespace` | String | `'default'` | Kubernetes namespace |
| `kubernetes_deployment` | String | `nil` | Deployment name to scale |
| `kubernetes_config_path` | String | `nil` | Path to kubeconfig (optional) |

## Common Configuration Examples

These examples show typical setups for different use cases. Copy and adapt them to your needs.

### Simple Setup (Single Worker, Heroku)

Ideal for small apps, side projects, or getting started:

```ruby
# config/initializers/solid_queue_autoscaler.rb
SolidQueueAutoscaler.configure do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'worker'

  config.min_workers = 1
  config.max_workers = 5

  # Scale up when queue backs up
  config.scale_up_queue_depth = 50
  config.scale_up_latency_seconds = 180  # 3 minutes

  # Scale down when queue is nearly empty
  config.scale_down_queue_depth = 5
  config.scale_down_latency_seconds = 30

  # Safety: only run in production
  config.dry_run = !Rails.env.production?
  config.enabled = Rails.env.production?
end
```

```yaml
# config/recurring.yml
autoscaler:
  class: SolidQueueAutoscaler::AutoscaleJob
  queue: autoscaler
  schedule: every 30 seconds
```

---

### Cost-Optimized Setup (Scale to Zero)

For apps with sporadic workloads where you want to minimize costs during idle periods. See the [Scale to Zero](#scale-to-zero) section for full details on how this works.

```ruby
SolidQueueAutoscaler.configure do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'worker'

  # Allow scaling to zero - no workers when idle
  config.min_workers = 0
  config.max_workers = 5

  # Scale up immediately when any job is queued
  config.scale_up_queue_depth = 1
  config.scale_up_latency_seconds = 60  # 1 minute

  # Scale down aggressively when empty
  config.scale_down_queue_depth = 0
  config.scale_down_latency_seconds = 10

  # Shorter cooldowns for faster response
  config.scale_up_cooldown_seconds = 30
  config.scale_down_cooldown_seconds = 300  # 5 min before scaling to zero

  config.enabled = Rails.env.production?
end
```

**⚠️ Note:** With `min_workers = 0`, there's cold-start latency (~30-90s) when the first job arrives. The autoscaler must run on a web dyno or separate always-on process, not on the workers themselves. See [Scale to Zero](#scale-to-zero) for details.

---

### E-Commerce / SaaS (Multiple Worker Types)

For apps with different job priorities (payments, notifications, reports):

```ruby
# Critical jobs - payments, webhooks, user-facing notifications
SolidQueueAutoscaler.configure(:critical_worker) do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'critical_worker'
  
  config.queues = ['critical', 'payments', 'webhooks']
  
  # Always have capacity, scale aggressively
  config.min_workers = 2
  config.max_workers = 10
  config.scale_up_queue_depth = 5
  config.scale_up_latency_seconds = 30
  
  # Short cooldowns for responsiveness
  config.cooldown_seconds = 60
  
  # High-priority autoscaler job
  config.job_queue = :autoscaler
  config.job_priority = 0
end

# Default jobs - emails, notifications, analytics
SolidQueueAutoscaler.configure(:default_worker) do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'worker'
  
  config.queues = ['default', 'mailers', 'analytics']
  
  # Standard capacity, moderate scaling
  config.min_workers = 1
  config.max_workers = 8
  config.scale_up_queue_depth = 100
  config.scale_up_latency_seconds = 300
  
  config.cooldown_seconds = 120
end

# Batch jobs - reports, exports, data processing
SolidQueueAutoscaler.configure(:batch_worker) do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'batch_worker'
  
  config.queues = ['batch', 'reports', 'exports']
  
  # Scale to zero when no batch jobs, scale up for any batch work
  config.min_workers = 0
  config.max_workers = 3
  config.scale_up_queue_depth = 1
  config.scale_down_queue_depth = 0
  
  # Long cooldowns - batch jobs take time
  config.cooldown_seconds = 300
end
```

```yaml
# config/recurring.yml
# Scale critical workers frequently
autoscaler_critical:
  class: SolidQueueAutoscaler::AutoscaleJob
  queue: autoscaler
  schedule: every 15 seconds
  args: [:critical_worker]

# Scale default workers normally
autoscaler_default:
  class: SolidQueueAutoscaler::AutoscaleJob
  queue: autoscaler
  schedule: every 30 seconds
  args: [:default_worker]

# Scale batch workers less frequently
autoscaler_batch:
  class: SolidQueueAutoscaler::AutoscaleJob
  queue: autoscaler
  schedule: every 60 seconds
  args: [:batch_worker]
```

---

### High-Volume API (Webhook Processing)

For apps processing many incoming webhooks or API callbacks:

```ruby
SolidQueueAutoscaler.configure do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'worker'

  config.queues = ['webhooks', 'callbacks', 'api_jobs']

  # Maintain baseline capacity
  config.min_workers = 2
  config.max_workers = 20

  # Proportional scaling - scale based on actual load
  config.scaling_strategy = :proportional
  config.scale_up_queue_depth = 50
  config.scale_up_latency_seconds = 60
  
  # Add 1 worker per 25 jobs over threshold
  config.scale_up_jobs_per_worker = 25
  # Add 1 worker per 30 seconds over latency threshold
  config.scale_up_latency_per_worker = 30
  
  # Scale down when under capacity
  config.scale_down_queue_depth = 10
  config.scale_down_jobs_per_worker = 50

  # Fast cooldowns for responsive scaling
  config.scale_up_cooldown_seconds = 30
  config.scale_down_cooldown_seconds = 120
  
  config.job_priority = 0  # Process autoscaler jobs first
end
```

---

### Data Processing / ETL Pipeline

For apps with heavy data processing, imports, or batch ETL jobs:

```ruby
SolidQueueAutoscaler.configure(:etl_worker) do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'etl_worker'

  config.queues = ['imports', 'exports', 'etl', 'data_sync']

  # Scale to zero when no work, burst when needed
  config.min_workers = 0
  config.max_workers = 10

  # Scale up as soon as work is queued
  config.scale_up_queue_depth = 1
  config.scale_up_latency_seconds = 120
  
  # Use fixed scaling for predictable behavior
  config.scaling_strategy = :fixed
  config.scale_up_increment = 2  # Add 2 workers at a time
  config.scale_down_decrement = 1

  # Long cooldowns - ETL jobs are long-running
  config.scale_up_cooldown_seconds = 120
  config.scale_down_cooldown_seconds = 600  # 10 minutes

  # Scale down only when truly idle
  config.scale_down_queue_depth = 0
  config.scale_down_latency_seconds = 0
end
```

---

### High-Availability Setup

For mission-critical apps requiring guaranteed capacity:

```ruby
SolidQueueAutoscaler.configure do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'worker'

  # Always maintain minimum capacity
  config.min_workers = 3
  config.max_workers = 15

  # Scale up proactively before queue backs up
  config.scale_up_queue_depth = 25
  config.scale_up_latency_seconds = 60

  # Conservative scale-down
  config.scale_down_queue_depth = 5
  config.scale_down_latency_seconds = 15

  # Longer cooldowns to prevent flapping
  config.cooldown_seconds = 180
  config.scale_down_cooldown_seconds = 300  # Extra cautious on scale-down

  # Record all events for monitoring
  config.record_events = true
  config.record_all_events = true  # Even no-change events
end
```

---

### Kubernetes Setup

For apps deployed on Kubernetes:

```ruby
SolidQueueAutoscaler.configure do |config|
  config.adapter = :kubernetes
  config.kubernetes_namespace = ENV.fetch('K8S_NAMESPACE', 'production')
  config.kubernetes_deployment = 'solid-queue-worker'
  
  # Optional: specify kubeconfig for local development
  # config.kubernetes_kubeconfig = '~/.kube/config'
  # config.kubernetes_context = 'my-cluster'

  config.min_workers = 2  # Minimum replicas
  config.max_workers = 20

  config.scale_up_queue_depth = 100
  config.scale_up_latency_seconds = 180
  
  config.scale_down_queue_depth = 10
  config.scale_down_latency_seconds = 30

  # K8s scaling can be faster than Heroku
  config.cooldown_seconds = 60
  
  config.enabled = Rails.env.production?
end
```

**Required RBAC configuration:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: solid-queue-autoscaler
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "deployments/scale"]
  verbs: ["get", "patch", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: solid-queue-autoscaler
  namespace: production
subjects:
- kind: ServiceAccount
  name: solid-queue-autoscaler
  namespace: production
roleRef:
  kind: Role
  name: solid-queue-autoscaler
  apiGroup: rbac.authorization.k8s.io
```

---

### Development / Testing Setup

For local development and CI environments:

```ruby
SolidQueueAutoscaler.configure do |config|
  config.adapter = :heroku
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'worker'

  config.min_workers = 1
  config.max_workers = 3

  config.scale_up_queue_depth = 10
  config.scale_up_latency_seconds = 60

  # IMPORTANT: Disable in development, use dry_run in staging
  case Rails.env
  when 'production'
    config.enabled = true
    config.dry_run = false
  when 'staging'
    config.enabled = true
    config.dry_run = true  # Log decisions but don't scale
  else
    config.enabled = false
  end

  # Verbose logging for debugging
  config.logger = Logger.new(STDOUT)
  config.logger.level = Rails.env.production? ? Logger::INFO : Logger::DEBUG
end
```

---

### Configuration Comparison

| Use Case | min | max | scale_up_depth | scale_up_latency | cooldown | strategy |
|----------|-----|-----|----------------|------------------|----------|----------|
| Simple/Starter | 1 | 5 | 50 | 180s | 120s | fixed |
| Cost-Optimized | 0 | 5 | 1 | 60s | 30s/300s | fixed |
| E-Commerce Critical | 2 | 10 | 5 | 30s | 60s | fixed |
| E-Commerce Default | 1 | 8 | 100 | 300s | 120s | fixed |
| Webhook Processing | 2 | 20 | 50 | 60s | 30s/120s | proportional |
| ETL/Batch | 0 | 10 | 1 | 120s | 120s/600s | fixed |
| High-Availability | 3 | 15 | 25 | 60s | 180s/300s | fixed |

## Configuring a High-Priority Queue for the Autoscaler

The autoscaler job should run reliably and quickly, even when your queues are backed up. By default, the autoscaler job runs on the `:autoscaler` queue. You can configure this and set up Solid Queue to prioritize it.

### Configure the Job Queue and Priority

In your initializer, set the queue and priority for the autoscaler job:

```ruby
SolidQueueAutoscaler.configure do |config|
  # Use a dedicated high-priority queue for the autoscaler
  config.job_queue = :autoscaler  # Default value
  
  # Or use an existing high-priority queue
  config.job_queue = :critical
  
  # Set job priority (lower = higher priority, processed first)
  # This works with queue backends that support job-level priority like Solid Queue
  config.job_priority = 0  # Highest priority
  
  # ... other config
end
```

For multi-worker configurations, each worker type can have its own queue and priority:

```ruby
SolidQueueAutoscaler.configure(:critical_worker) do |config|
  config.job_queue = :autoscaler_critical
  config.job_priority = 0  # Highest priority for critical worker scaling
  # ... other config
end

SolidQueueAutoscaler.configure(:default_worker) do |config|
  config.job_queue = :autoscaler_default
  config.job_priority = 10  # Lower priority for default worker scaling
  # ... other config
end
```

### Configure Solid Queue to Prioritize the Autoscaler

In your `config/solid_queue.yml`, ensure the autoscaler queue is processed by a dedicated worker or listed first in the queue order:

```yaml
# Option 1: Dedicated dispatcher/worker for autoscaler (recommended)
production:
  dispatchers:
    - polling_interval: 1
      batch_size: 500
      concurrency_maintenance_interval: 30

  workers:
    # Dedicated worker for autoscaler - always responsive
    - queues: [autoscaler]
      threads: 1
      processes: 1
      polling_interval: 0.5  # Check frequently
    
    # Main workers for business logic
    - queues: [critical, default, mailers]
      threads: 5
      processes: 2
      polling_interval: 1
```

```yaml
# Option 2: Include autoscaler first in queue list (simpler)
production:
  workers:
    - queues: [autoscaler, critical, default, mailers]
      threads: 5
      processes: 2
```

Solid Queue processes queues in order, so listing `autoscaler` first ensures those jobs are picked up before others.

### Why This Matters

- **Responsiveness**: When your queues are backed up, you want the autoscaler to scale up workers quickly
- **Reliability**: A dedicated queue prevents autoscaler jobs from waiting behind thousands of business jobs
- **Isolation**: Separating autoscaler jobs makes monitoring and debugging easier

## Usage

### Running as a Solid Queue Recurring Job (Recommended)

> ⚠️ **IMPORTANT**: The `queue:` setting in `recurring.yml` **overrides** the `config.job_queue` setting!
> If you omit `queue:` in your recurring.yml, the job will go to the `default` queue, NOT your configured queue.
> Always ensure your `recurring.yml` queue matches your `config.job_queue` setting.

Add to your `config/recurring.yml`:

```yaml
# Single worker configuration
autoscaler:
  class: SolidQueueAutoscaler::AutoscaleJob
  queue: autoscaler  # ⚠️ REQUIRED: Must match config.job_queue!
  schedule: every 30 seconds

# Or for multi-worker: scale all workers
autoscaler_all:
  class: SolidQueueAutoscaler::AutoscaleJob
  queue: autoscaler  # ⚠️ REQUIRED!
  schedule: every 30 seconds
  args:
    - :all

# Or scale specific worker types on different schedules
autoscaler_critical:
  class: SolidQueueAutoscaler::AutoscaleJob
  queue: autoscaler  # ⚠️ REQUIRED!
  schedule: every 15 seconds
  args:
    - :critical_worker

autoscaler_default:
  class: SolidQueueAutoscaler::AutoscaleJob
  queue: autoscaler  # ⚠️ REQUIRED!
  schedule: every 60 seconds
  args:
    - :default_worker
```

> **Note on multiple worker dynos**: SolidQueue's recurring jobs are processed by the **dispatcher** process,
> not workers. If each of your worker dynos runs its own dispatcher (which is the default on Heroku),
> each dyno will try to enqueue the recurring job. To prevent duplicate enqueuing:
> 1. Run a single dedicated dispatcher dyno, OR
> 2. Configure workers to NOT run the dispatcher (set `dispatchers: []` in their solid_queue.yml)

### Running via Rake Tasks

```bash
# Scale the default worker
bundle exec rake solid_queue_autoscaler:scale

# Scale a specific worker type
WORKER=critical_worker bundle exec rake solid_queue_autoscaler:scale

# Scale all configured workers
bundle exec rake solid_queue_autoscaler:scale_all

# List all registered worker configurations
bundle exec rake solid_queue_autoscaler:workers

# View metrics for default worker
bundle exec rake solid_queue_autoscaler:metrics

# View metrics for specific worker
WORKER=critical_worker bundle exec rake solid_queue_autoscaler:metrics

# View current formation
bundle exec rake solid_queue_autoscaler:formation

# Check cooldown status
bundle exec rake solid_queue_autoscaler:cooldown

# Reset cooldowns
bundle exec rake solid_queue_autoscaler:reset_cooldown
```

### Running Programmatically

```ruby
# Scale the default worker
result = SolidQueueAutoscaler.scale!

# Scale a specific worker type
result = SolidQueueAutoscaler.scale!(:critical_worker)

# Scale all configured workers
results = SolidQueueAutoscaler.scale_all!

# Get metrics for a specific worker
metrics = SolidQueueAutoscaler.metrics(:critical_worker)
puts "Queue depth: #{metrics.queue_depth}"
puts "Latency: #{metrics.oldest_job_age_seconds}s"

# Get current worker count
workers = SolidQueueAutoscaler.current_workers(:default_worker)
puts "Current workers: #{workers}"

# List all registered workers
SolidQueueAutoscaler.registered_workers
# => [:critical_worker, :default_worker, :batch_worker]

# Get configuration for a specific worker
config = SolidQueueAutoscaler.config(:critical_worker)
```

## How It Works

### Metrics Collection

The autoscaler queries Solid Queue's PostgreSQL tables to collect:

- **Queue depth**: Count of jobs in `solid_queue_ready_executions`
- **Oldest job age**: Time since oldest job was enqueued (latency)
- **Throughput**: Jobs completed in the last minute
- **Active workers**: Workers with recent heartbeats
- **Per-queue breakdown**: Job counts by queue name

When `queues` is configured, metrics are filtered to only those queues.

### Decision Logic

**Scale Up** when ANY of these conditions are met:
- Queue depth >= `scale_up_queue_depth`
- Oldest job age >= `scale_up_latency_seconds`

**Scale Down** when ALL of these conditions are met:
- Queue depth <= `scale_down_queue_depth`
- Oldest job age <= `scale_down_latency_seconds`
- OR queue is completely idle (no pending or claimed jobs)

**No Change** when:
- Already at min/max workers
- Within cooldown period
- Metrics are in normal range

### Scaling Strategies

**Fixed Strategy** (default): Adds/removes a fixed number of workers per scaling event.

```ruby
config.scaling_strategy = :fixed
config.scale_up_increment = 2    # Add 2 workers when scaling up
config.scale_down_decrement = 1  # Remove 1 worker when scaling down
```

**Proportional Strategy**: Scales based on how far over/under thresholds you are.

```ruby
config.scaling_strategy = :proportional
config.scale_up_jobs_per_worker = 50      # Add 1 worker per 50 jobs over threshold
config.scale_up_latency_per_worker = 60   # Add 1 worker per 60s over threshold
```

### Singleton Execution

PostgreSQL advisory locks ensure only one autoscaler instance runs at a time, even across multiple dynos/pods. Each worker configuration gets its own lock key, so different worker types can scale simultaneously.

### Cooldowns

After each scaling event, a cooldown period prevents additional scaling:
- Prevents "flapping" between states
- Gives the platform time to spin up new workers
- Allows queue to stabilize after scaling

Cooldowns are tracked per-worker type, so scaling one worker doesn't block scaling another.

## Environment Variables

### Heroku

| Variable | Description | Required |
|----------|-------------|----------|
| `HEROKU_API_KEY` | Heroku Platform API token | Yes |
| `HEROKU_APP_NAME` | Name of your Heroku app | Yes |

### Kubernetes

| Variable | Description | Required |
|----------|-------------|----------|
| `KUBERNETES_NAMESPACE` | Kubernetes namespace | No (defaults to 'default') |

## Dry Run Mode

Test the autoscaler without making actual changes:

```ruby
SolidQueueAutoscaler.configure do |config|
  config.dry_run = true
  # ... other config
end
```

In dry-run mode, all decisions are logged but no platform API calls are made.

## Dashboard

The optional dashboard provides a web UI for monitoring the autoscaler:

### Features

- **Overview Dashboard**: Real-time metrics, worker status, and recent events
- **Workers View**: Detailed status for each worker type with configuration and cooldowns
- **Events Log**: Historical record of all scaling decisions with filtering
- **Manual Scaling**: Trigger scale operations directly from the UI

### Setup

1. Generate the dashboard migration:

```bash
rails generate solid_queue_autoscaler:dashboard
rails db:migrate
```

2. Mount the engine in `config/routes.rb`:

```ruby
authenticate :user, ->(u) { u.admin? } do
  mount SolidQueueAutoscaler::Dashboard::Engine => "/autoscaler"
end
```

3. Visit `/autoscaler` in your browser

### Event Recording

By default, all scaling events are recorded to the database. Configure in your initializer:

```ruby
SolidQueueAutoscaler.configure do |config|
  # Record scale_up, scale_down, skipped, and error events (default: true)
  config.record_events = true
  
  # Also record no_change events (verbose, default: false)
  config.record_all_events = false
end
```

### Rake Tasks for Events

```bash
# View recent scale events
bundle exec rake solid_queue_autoscaler:events

# View events for a specific worker
WORKER=critical_worker bundle exec rake solid_queue_autoscaler:events

# Cleanup old events (default: keep 30 days)
bundle exec rake solid_queue_autoscaler:cleanup_events
KEEP_DAYS=7 bundle exec rake solid_queue_autoscaler:cleanup_events
```

## Troubleshooting

### "Could not acquire advisory lock"

Another autoscaler instance is currently running. This is expected behavior — only one instance should run at a time per worker type.

**If you believe no other instance is running:**

```ruby
# Check for stale advisory locks
ActiveRecord::Base.connection.execute(<<~SQL)
  SELECT * FROM pg_locks WHERE locktype = 'advisory'
SQL

# Force release a stuck lock (use with caution!)
lock_key = SolidQueueAutoscaler.config.lock_key
lock_id = Zlib.crc32(lock_key) & 0x7FFFFFFF
ActiveRecord::Base.connection.execute("SELECT pg_advisory_unlock(#{lock_id})")
```

### "Cooldown active"

A recent scaling event triggered the cooldown. Wait for the cooldown to expire or adjust `cooldown_seconds`.

```ruby
# Check cooldown status
bundle exec rake solid_queue_autoscaler:cooldown

# Reset cooldowns (for testing only)
SolidQueueAutoscaler::CooldownTracker.reset!
```

### Workers not scaling

1. Check that `enabled` is `true`
2. Verify platform credentials are set correctly
3. Check metrics with `rake solid_queue_autoscaler:metrics`
4. Enable dry-run to see what decisions would be made
5. Check the logs for error messages

**Debug with a manual scale attempt:**

```ruby
# Check configuration
config = SolidQueueAutoscaler.config
puts "Enabled: #{config.enabled?}"
puts "Dry Run: #{config.dry_run?}"
puts "API Key Set: #{config.heroku_api_key.present?}"

# Check current metrics
metrics = SolidQueueAutoscaler.metrics
puts "Queue depth: #{metrics.queue_depth}"
puts "Latency: #{metrics.oldest_job_age_seconds}s"

# Try a manual scale
result = SolidQueueAutoscaler.scale!
puts result.decision.inspect if result.decision
puts result.skipped_reason if result.skipped?
puts result.error if result.error
```

### Workers not scaling down

Scale-down requires **ALL** conditions to be met:

```ruby
metrics = SolidQueueAutoscaler.metrics
config = SolidQueueAutoscaler.config

puts "Queue depth: #{metrics.queue_depth} (threshold: <= #{config.scale_down_queue_depth})"
puts "Latency: #{metrics.oldest_job_age_seconds}s (threshold: <= #{config.scale_down_latency_seconds}s)"
puts "Claimed jobs: #{metrics.claimed_jobs}"  # Must be 0 for idle scale-down
puts "Current workers: #{SolidQueueAutoscaler.current_workers}"
puts "Min workers: #{config.min_workers}"  # Can't scale below this
```

### Heroku API errors

**401 Unauthorized:**

```bash
# Check if API key is valid
heroku authorizations

# Create a new authorization
heroku authorizations:create -d "Solid Queue Autoscaler"
```

**404 Not Found:**

```bash
# Verify app name
heroku apps
heroku apps:info -a $HEROKU_APP_NAME
```

**429 Rate Limited:**

Increase cooldown to reduce API calls:

```ruby
config.cooldown_seconds = 180  # 3 minutes instead of default 2
```

### Kubernetes authentication issues

1. Ensure the service account has permissions to patch deployments
2. Check namespace is correct
3. Verify deployment name matches exactly

**Check RBAC permissions:**

```yaml
# Required RBAC rules for the autoscaler service account
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: solid-queue-autoscaler
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "deployments/scale"]
  verbs: ["get", "patch", "update"]
```

### AutoscaleJob not running

**Check recurring.yml configuration:**

```yaml
# config/recurring.yml
autoscaler:
  class: SolidQueueAutoscaler::AutoscaleJob
  queue: autoscaler
  schedule: every 30 seconds
```

**Ensure a worker processes the autoscaler queue:**

```yaml
# config/solid_queue.yml
workers:
  - queues: [autoscaler]  # Must include autoscaler queue
    threads: 1
```

**Test manual enqueue:**

```ruby
SolidQueueAutoscaler::AutoscaleJob.perform_later
```

### Multi-worker configuration issues

**"Unknown worker: :my_worker":**

Ensure you've configured the worker before referencing it:

```ruby
# Configure the worker first
SolidQueueAutoscaler.configure(:my_worker) do |config|
  config.heroku_api_key = ENV['HEROKU_API_KEY']
  config.heroku_app_name = ENV['HEROKU_APP_NAME']
  config.process_type = 'my_worker'
end

# Then reference it
SolidQueueAutoscaler.scale!(:my_worker)
```

**List all registered workers:**

```ruby
SolidQueueAutoscaler.registered_workers
# => [:default, :critical_worker, :batch_worker]
```

### Database/Migration issues

**"relation 'solid_queue_autoscaler_state' does not exist":**

```bash
rails generate solid_queue_autoscaler:migration
rails db:migrate
```

**"relation 'solid_queue_ready_executions' does not exist":**

Solid Queue tables are missing. Run Solid Queue migrations:

```bash
rails solid_queue:install:migrations
rails db:migrate
```

**Multi-database setup (Solid Queue on separate database):**

The autoscaler automatically detects `SolidQueue::Record.connection`. If auto-detection fails:

```ruby
SolidQueueAutoscaler.configure do |config|
  config.database_connection = SolidQueue::Record.connection
end
```

### Dashboard not loading

**404 when visiting /autoscaler:**

Ensure the engine is mounted in `config/routes.rb`:

```ruby
mount SolidQueueAutoscaler::Dashboard::Engine => "/autoscaler"
```

**"ActionView::MissingTemplate" errors:**

Run the dashboard generator:

```bash
rails generate solid_queue_autoscaler:dashboard
rails db:migrate
```

### Wrong process type being scaled

```ruby
# Check what process type is configured
puts SolidQueueAutoscaler.config.process_type

# Verify it matches your Procfile
# Procfile:
# web: bundle exec puma -C config/puma.rb
# worker: bundle exec rake solid_queue:start  # <- This is "worker"
```

### Scaling too aggressively or too slowly

**Scaling up too often (flapping):**

```ruby
config.cooldown_seconds = 180           # Increase cooldown
config.scale_up_cooldown_seconds = 120  # Or set scale-up specific cooldown
config.scale_up_queue_depth = 200       # Increase threshold
```

**Not scaling up fast enough:**

```ruby
config.scale_up_queue_depth = 50        # Lower threshold
config.scale_up_latency_seconds = 120   # Trigger on 2 min latency
config.cooldown_seconds = 60            # Reduce cooldown
config.scaling_strategy = :proportional # Scale based on load, not fixed increment
config.scale_up_jobs_per_worker = 25    # More workers per jobs over threshold
```

**Not scaling down:**

```ruby
config.scale_down_queue_depth = 5       # More aggressive scale-down threshold  
config.scale_down_latency_seconds = 10  # Tighter latency requirement
config.min_workers = 0                  # Allow scaling to zero (if appropriate)
```

### Debugging tips

**Enable debug logging:**

```ruby
SolidQueueAutoscaler.configure do |config|
  config.logger = Logger.new(STDOUT)
  config.logger.level = Logger::DEBUG
end
```

**Simulate a scaling decision without making changes:**

```ruby
metrics = SolidQueueAutoscaler.metrics
workers = SolidQueueAutoscaler.current_workers
engine = SolidQueueAutoscaler::DecisionEngine.new(config: SolidQueueAutoscaler.config)
decision = engine.decide(metrics: metrics, current_workers: workers)

puts "Action: #{decision.action}"    # :scale_up, :scale_down, or :no_change
puts "From: #{decision.from} -> To: #{decision.to}"
puts "Reason: #{decision.reason}"
```

**Run diagnostics:**

```bash
bundle exec rake solid_queue_autoscaler:metrics
bundle exec rake solid_queue_autoscaler:formation
bundle exec rake solid_queue_autoscaler:cooldown
```

For more detailed troubleshooting, see [docs/troubleshooting.md](docs/troubleshooting.md).

## Architecture Notes

This gem acts as a **control plane** for Solid Queue:

- **External to workers**: The autoscaler must not depend on the workers it's scaling
- **Singleton**: Advisory locks ensure only one instance runs globally per worker type
- **Dedicated queue**: Runs on its own queue to avoid competing with business jobs
- **Conservative**: Defaults to gradual scaling with cooldowns
- **Multi-tenant**: Each worker configuration is independent

## License

MIT License. See [LICENSE.txt](LICENSE.txt) for details.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for development setup and contribution guidelines.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/my-feature`)
3. Write tests for your changes
4. Ensure all tests pass (`bundle exec rspec`)
5. Ensure RuboCop passes (`bundle exec rubocop`)
6. Submit a pull request

## Links

- [GitHub Repository](https://github.com/reillyse/solid_queue_autoscaler)
- [RubyGems](https://rubygems.org/gems/solid_queue_autoscaler)
- [Changelog](CHANGELOG.md)

