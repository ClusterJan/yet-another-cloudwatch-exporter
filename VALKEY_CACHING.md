# Valkey Caching Integration for AWS Tagging API

## Overview

This document describes the integration of **Valkey caching** into YACE (Yet Another CloudWatch Exporter) to reduce excessive AWS GetResources API calls. By caching the raw AWS tagging API responses for 10 minutes, YACE can serve multiple requests within that period without hitting AWS, while still maintaining the current tag value filtering logic.

## Problem Statement

YACE makes too many calls to the AWS Resource Groups Tagging API (`GetResources`). Each discovery job queries the API with tag key filters, but the current logic calls AWS for every combination of:
- Region
- Resource filters
- Tag filter keys

This results in redundant API calls when the same combination is queried multiple times or by multiple jobs.

## Solution Architecture

### Cache Strategy

1. **What is cached**: Raw AWS GetResources API responses (ARN + tags map)
2. **Cache key**: `yace:tagging:<region>:<sorted_filters>:<sorted_tag_keys>`
3. **TTL**: 10 minutes (configurable constant)
4. **Cache backend**: Valkey (valkey-go client)

### How It Works

```
Request comes in
    ↓
Build cache key from region + filters + tag keys
    ↓
Check Valkey cache
    ├─→ Cache HIT: Return cached data
    │       ↓
    │   Apply local tag filtering (FilterThroughTags)
    │       ↓
    │   Return filtered resources
    │
    └─→ Cache MISS: Call AWS API
            ↓
        Store raw response in Valkey (10-min TTL)
            ↓
        Apply local tag filtering (FilterThroughTags)
            ↓
        Return filtered resources
```

### Key Design Decisions

1. **Local Filtering**: AWS is called with only tag **keys** (not values). Tag value filtering happens locally in YACE using `FilterThroughTags()`. This means:
   - Reduces AWS API response size
   - Allows caching of the raw response
   - Preserves existing filtering behavior

2. **Sorted Cache Keys**: Resource filters and tag filter keys are sorted before creating the cache key. This ensures:
   - Same cache hit regardless of order
   - Consistent key generation
   - Deterministic caching behavior

3. **Optional Cache**: Cache is completely optional and disabled by default:
   - No Valkey address configured = cache disabled
   - Cache failures gracefully fallback to AWS API
   - Existing behavior preserved for users without Valkey

4. **10-Minute TTL**: Provides good balance between:
   - Reducing API calls (most scrape intervals are longer)
   - Keeping data relatively fresh
   - Allowing periodic updates

## Implementation Details

### Files Created

#### `pkg/clients/tagging/cache.go`

Defines caching abstraction and Valkey implementation:

```go
// Cache interface - abstraction for any caching backend
type Cache interface {
    Get(ctx context.Context, key string) ([]ResourceTagMappingCache, error)
    Set(ctx context.Context, key string, mappings []ResourceTagMappingCache, ttl time.Duration) error
    Close()
}

// ValkeyCache - implements Cache using valkey-go
type ValkeyCache struct {
    client valkey.Client
    logger *slog.Logger
}

// ValkeyConfig - configuration for Valkey connection
type ValkeyConfig struct {
    Address  string // e.g., "localhost:6379"
    Username string // optional
    Password string // optional
    DB       int    // database number
    TLS      bool   // TLS support
}

// ResourceTagMappingCache - represents cached AWS response
type ResourceTagMappingCache struct {
    ResourceARN string            `json:"resource_arn"`
    Tags        map[string]string `json:"tags"`
}

// BuildCacheKey - creates deterministic cache keys
func BuildCacheKey(region string, resourceFilters []string, tagFilterKeys []string) string {
    // Returns: "yace:tagging:<region>:<sorted_filters>:<sorted_tag_keys>"
}
```

Key features:
- Generic `Cache` interface allows future implementations (Redis, Memcached, etc.)
- `NewValkeyCache()` validates connection on creation
- JSON serialization for cache storage
- Sorted keys for consistency
- 10-minute default TTL

### Files Modified

#### `pkg/clients/tagging/v2/client.go`

Updated tagging client to support caching:

```go
type client struct {
    // ... existing fields ...
    cache tagging.Cache  // Optional cache backend
}

// NewClientWithCache - creates client with optional cache
func NewClientWithCache(
    logger *slog.Logger,
    taggingAPI *resourcegroupstaggingapi.Client,
    // ... other params ...
    cache tagging.Cache,  // Can be nil
) tagging.Client

// GetResources - modified to use cache
func (c client) GetResources(ctx context.Context, job model.DiscoveryJob, region string) ([]*model.TaggedResource, error) {
    // 1. Build cache key
    // 2. Try cache.Get(key)
    // 3. If cache hit: use cached data + local filtering
    // 4. If cache miss: call AWS API + cache.Set(key, data) + local filtering
    // 5. Return filtered resources
}
```

Changes:
- Added optional `cache` field to client struct
- New `NewClientWithCache()` function for cache support
- `GetResources()` checks cache before AWS API
- Caches raw AWS response with 10-minute TTL
- Applies local tag filtering on cached and non-cached responses
- Helper methods `getFromCache()` and `setToCache()` handle nil cache safely

#### `pkg/clients/v2/factory.go`

Updated factory to support cache creation:

```go
type CachingFactory struct {
    // ... existing fields ...
    taggingCache tagging.Cache  // Optional cache shared across all clients
}

// NewFactoryWithCache - creates factory with optional cache
func NewFactoryWithCache(
    logger *slog.Logger,
    jobsCfg model.JobsConfig,
    fips bool,
    taggingCache tagging.Cache,  // Can be nil
) (*CachingFactory, error)

// GetTaggingClient - passes cache to client
func (c *CachingFactory) GetTaggingClient(...) tagging.Client {
    // Uses NewClientWithCache with shared cache
}
```

Changes:
- Added `taggingCache` field to factory
- New `NewFactoryWithCache()` function
- Updated `GetTaggingClient()` to use `NewClientWithCache()`
- Updated `Refresh()` to use cache

#### `cmd/yace/main.go`

Added CLI configuration and cache initialization:

```go
var (
    // ... existing variables ...
    valkeyAddress  string
    valkeyUsername string
    valkeyPassword string
    valkeyDB       int
)

// CLI flags added:
// --valkey.address     (env: VALKEY_ADDRESS)
// --valkey.username    (env: VALKEY_USERNAME)
// --valkey.password    (env: VALKEY_PASSWORD)
// --valkey.db          (env: VALKEY_DB)

// createValkeyCache - helper to create cache if configured
func createValkeyCache(ctx context.Context, logger *slog.Logger) (tagging.Cache, error) {
    if valkeyAddress == "" {
        return nil, nil  // Cache disabled
    }
    return tagging.NewValkeyCache(ctx, logger, tagging.ValkeyConfig{
        Address:  valkeyAddress,
        Username: valkeyUsername,
        Password: valkeyPassword,
        DB:       valkeyDB,
    })
}

// startScraper - updated to create and use cache
func startScraper(c *cli.Context) error {
    // ... load config ...
    
    // Create Valkey cache if configured
    valkeyCache, err := createValkeyCache(context.Background(), logger)
    // ... handle error ...
    
    // Create factory with cache
    if valkeyCache != nil {
        cache = v2.NewFactoryWithCache(logger, jobsCfg, fips, valkeyCache)
    } else {
        cache = v2.NewFactory(logger, jobsCfg, fips)
    }
}
```

Changes:
- Added 4 new CLI flags for Valkey configuration
- Added `createValkeyCache()` helper function
- Updated `startScraper()` to create cache
- Updated `/reload` endpoint to handle cache
- Graceful fallback if cache creation fails

#### `go.mod`

Added dependency:
```
require (
    github.com/valkey-io/valkey-go v1.0.69
)
```

## Usage

### Without Caching (Default)

```bash
# No Valkey configuration = cache disabled
yace
```

YACE behaves exactly as before. No changes to existing behavior.

### With Valkey Caching

#### Option 1: Environment Variables

```bash
export VALKEY_ADDRESS="localhost:6379"
export VALKEY_USERNAME="myuser"          # optional
export VALKEY_PASSWORD="mypassword"      # optional
export VALKEY_DB="0"                     # optional

yace
```

#### Option 2: CLI Flags

```bash
yace \
  --valkey.address localhost:6379 \
  --valkey.username myuser \
  --valkey.password mypassword \
  --valkey.db 0
```

#### Option 3: Docker with Environment

```dockerfile
FROM yet-another-cloudwatch-exporter

ENV VALKEY_ADDRESS="valkey:6379"
ENV VALKEY_PASSWORD="secret"
```

### Verifying Cache Is Working

Check logs for cache operations:

```
# Cache initialization
INFO Connected to Valkey cache address=localhost:6379

# Successful cache miss
DEBUG Cache miss key=yace:tagging:us-east-1:aws:ec2:env,team

# Successful cache hit
DEBUG Cache hit key=yace:tagging:us-east-1:aws:ec2:env,team count=42

# Cache operation
DEBUG Cache set key=yace:tagging:us-east-1:aws:ec2:env,team count=42 ttl=10m0s
```

## Performance Impact

### Before Caching

```
Scrape interval: 5 minutes
Discovery jobs: 3
Regions: 2
Unique filter combinations: 1

AWS GetResources calls per scrape: 3 jobs × 2 regions = 6 calls
AWS GetResources calls per 10 minutes: 12 calls (2 scrapes)
AWS GetResources calls per hour: 72 calls
```

### After Caching

```
Same setup as above

First scrape (time 0): 6 AWS calls
Second scrape (time 5min): 0 AWS calls (all cached)
Third scrape (time 10min): 6 AWS calls (cache expired)

AWS GetResources calls per 10 minutes: 6 calls (1 scrape)
AWS GetResources calls per hour: 36 calls (50% reduction)
```

**Real-world impact varies based on:**
- Number of scrape intervals within 10 minutes
- Number of unique filter combinations
- Cache hit rate

## Error Handling

### Cache Creation Failures

```go
valkeyCache, err := createValkeyCache(ctx, logger)
if err != nil {
    logger.Warn("Failed to create valkey cache, continuing without caching", "error", err)
}
// Continues with or without cache
```

If Valkey is unreachable during startup, YACE logs a warning and continues without caching.

### Cache Operation Failures

```go
// In GetResources
cachedMappings, cacheErr := c.getFromCache(ctx, cacheKey)
if cacheErr != nil {
    c.logger.Warn("Failed to get from cache, falling back to AWS API", "error", cacheErr)
}
// Falls back to AWS API
```

If cache operations fail during runtime:
- Cache misses: Falls back to AWS API
- Cache write failures: Logged as warning, continues operation
- No impact on resource discovery

## Configuration Examples

### Development Environment

```bash
# Local Valkey with default settings
VALKEY_ADDRESS="localhost:6379" yace
```

### Production with Authentication

```bash
# Valkey cluster with authentication
export VALKEY_ADDRESS="valkey-cluster.example.com:6379"
export VALKEY_USERNAME="yace-user"
export VALKEY_PASSWORD="$(cat /run/secrets/valkey_password)"
export VALKEY_DB="1"

yace --config.file production.yml
```

### Kubernetes Deployment

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: yace-config
data:
  config.yml: |
    # ... yace config ...

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yace
spec:
  template:
    spec:
      containers:
      - name: yace
        image: yet-another-cloudwatch-exporter:latest
        env:
        - name: VALKEY_ADDRESS
          value: "valkey-service:6379"
        - name: VALKEY_PASSWORD
          valueFrom:
            secretKeyRef:
              name: valkey-credentials
              key: password
        - name: VALKEY_DB
          value: "0"
        volumeMounts:
        - name: config
          mountPath: /etc/yace
      volumes:
      - name: config
        configMap:
          name: yace-config
```

## Testing

### Running Tests

```bash
# Test all packages
go test ./...

# Test specific package
go test ./pkg/clients/tagging/...

# Run with verbose output
go test -v ./pkg/clients/tagging/...

# Run with coverage
go test -cover ./...
```

### Manual Testing

1. **Start Valkey**
   ```bash
   docker run -p 6379:6379 valkey/valkey:latest
   ```

2. **Start YACE with caching**
   ```bash
   VALKEY_ADDRESS="localhost:6379" yace
   ```

3. **Monitor cache**
   ```bash
   # In another terminal
   valkey-cli
   > KEYS yace:tagging:*
   > GET yace:tagging:us-east-1:aws:ec2:env,team
   ```

4. **Check logs**
   ```bash
   # Look for cache hit/miss messages
   tail -f yace.log | grep -i cache
   ```

## Troubleshooting

### Cache Not Being Used

**Symptom**: No cache-related log messages

**Solution**:
1. Check if `VALKEY_ADDRESS` environment variable is set
2. Verify CLI flag: `--valkey.address`
3. Check logs for "Connected to Valkey cache"

### Valkey Connection Errors

**Symptom**: `Failed to create valkey cache: failed to connect to valkey`

**Solution**:
1. Verify Valkey server is running: `telnet localhost 6379`
2. Check address/port: `VALKEY_ADDRESS="host:port"`
3. Verify credentials if using authentication
4. Check firewall rules

### Cache Entries Not Expiring

**Symptom**: Cache growing indefinitely

**Solution**:
1. Verify Valkey is running properly
2. Check Valkey memory: `valkey-cli INFO memory`
3. Review Valkey maxmemory policy
4. Restart Valkey if necessary

### Performance Not Improving

**Symptom**: Cache not reducing AWS API calls as expected

**Solution**:
1. Check cache hit/miss logs
2. Verify scrape interval is less than 10 minutes
3. Check if filter combinations are changing frequently
4. Review cache key generation: `yace:tagging:<region>:<filters>:<tags>`

## Future Enhancements

1. **Configurable TTL**: Allow users to set custom cache TTL
2. **Cache Statistics**: Expose metrics about cache hits/misses
3. **Alternative Backends**: Support Redis, Memcached, etc.
4. **Cache Warming**: Pre-populate cache on startup
5. **Cache Invalidation**: Manual cache clearing endpoint
6. **Multi-region Optimization**: Smart cache key generation for multi-region deployments

## Contributing

When modifying cache-related code:

1. Update `pkg/clients/tagging/cache.go` for cache interface changes
2. Update `pkg/clients/tagging/v2/client.go` for caching logic
3. Update tests to verify cache behavior
4. Update this documentation
5. Add integration tests if modifying cache backends

## References

- [Valkey Documentation](https://valkey.io/docs)
- [valkey-go Library](https://github.com/valkey-io/valkey-go)
- [AWS Resource Groups Tagging API](https://docs.aws.amazon.com/resourcegroupstagging/latest/APIReference/)
- [YACE Documentation](./README.md)
