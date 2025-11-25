# Thanos Receive Multi-Shard Configuration

This Helm chart supports deploying Thanos Receive with multiple shards, where each shard has its own StatefulSet, S3 bucket, and secret configuration.

## Overview

Multi-shard mode allows you to:
- **Scale horizontally** across multiple independent shards
- **Isolate data** by using separate S3 buckets per shard
- **Customize resources** per shard (CPU, memory, storage, node selectors, etc.)
- **Simplify operations** with automatic hashring configuration

## Architecture

### Multi-Shard Mode
When `receive.shards` is configured:
- Each shard gets its own StatefulSet (e.g., `thanos-receive-shard-0`, `thanos-receive-shard-1`)
- Each shard has dedicated services (regular + headless)
- A common service (`thanos-receive`) load-balances across all shards for remote write
- The hashring includes endpoints from all shards across all replicas
- Each shard uses its own S3 secret for object storage

### Legacy Single-Shard Mode
When `receive.shards` is empty or not defined:
- Single StatefulSet with `replicaCount` replicas
- Uses `global.objstore` for object storage
- Backwards compatible with existing configurations

## Configuration

### Basic Multi-Shard Setup

```yaml
receive:
  enabled: true

  shards:
    - name: shard-0
      replicaCount: 3
      objstore:
        secretName: thanos-receive-shard-0-objstore
        secretKey: objstore.yml

    - name: shard-1
      replicaCount: 3
      objstore:
        secretName: thanos-receive-shard-1-objstore
        secretKey: objstore.yml

    - name: shard-2
      replicaCount: 3
      objstore:
        secretName: thanos-receive-shard-2-objstore
        secretKey: objstore.yml
```

### Per-Shard Resource Overrides

You can customize resources, node placement, and other settings per shard:

```yaml
receive:
  shards:
    - name: shard-0
      replicaCount: 3
      objstore:
        secretName: thanos-receive-shard-0-objstore
        secretKey: objstore.yml
      # Per-shard resource overrides
      resources:
        requests:
          memory: 4Gi
          cpu: 2
        limits:
          memory: 8Gi
          cpu: 4
      # Per-shard node selector
      nodeSelector:
        disktype: ssd
        zone: us-east-1a
      # Per-shard affinity rules
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/shard
                    operator: In
                    values:
                      - shard-0
              topologyKey: kubernetes.io/hostname
      # Per-shard storage configuration
      persistence:
        enabled: true
        size: 100Gi
        storageClass: fast-ssd
```

### Common Receive Configuration

Settings that apply to all shards:

```yaml
receive:
  # Common configuration
  tenancyHeader: "THANOS-TENANT"

  tsdb:
    retention: 24h
    walCompression: true

  hashrings:
    autogen:
      enabled: true
      name: default

  service:
    type: ClusterIP
    grpcPort: 10901
    httpPort: 10902
    remoteWritePort: 10908

  # Default resources (can be overridden per shard)
  resources:
    requests:
      memory: 2Gi
      cpu: 1
```

## Creating Secrets for Each Shard

Each shard requires its own secret with object storage configuration:

### S3 Example

```bash
# Shard 0
kubectl create secret generic thanos-receive-shard-0-objstore \
  --from-literal=objstore.yml='
type: S3
config:
  bucket: thanos-receive-shard-0
  endpoint: s3.amazonaws.com
  region: us-east-1
  access_key: YOUR_ACCESS_KEY_0
  secret_key: YOUR_SECRET_KEY_0
'

# Shard 1
kubectl create secret generic thanos-receive-shard-1-objstore \
  --from-literal=objstore.yml='
type: S3
config:
  bucket: thanos-receive-shard-1
  endpoint: s3.amazonaws.com
  region: us-east-1
  access_key: YOUR_ACCESS_KEY_1
  secret_key: YOUR_SECRET_KEY_1
'
```

### GCS Example

```bash
# Shard 0
kubectl create secret generic thanos-receive-shard-0-objstore \
  --from-literal=objstore.yml='
type: GCS
config:
  bucket: thanos-receive-shard-0
  service_account: /path/to/sa-0.json
'

# Shard 1
kubectl create secret generic thanos-receive-shard-1-objstore \
  --from-literal=objstore.yml='
type: GCS
config:
  bucket: thanos-receive-shard-1
  service_account: /path/to/sa-1.json
'
```

## Deployment

### Install with Multi-Shard Configuration

```bash
helm install thanos . \
  --values examples/receive-multi-shard-values.yaml \
  --namespace monitoring
```

### Verify Deployment

Check that all StatefulSets are created:

```bash
kubectl get statefulsets -n monitoring | grep receive
# Expected output:
# thanos-receive-shard-0   3/3     ...
# thanos-receive-shard-1   3/3     ...
# thanos-receive-shard-2   3/3     ...
```

Check services:

```bash
kubectl get services -n monitoring | grep receive
# Expected output:
# thanos-receive                  ClusterIP   ...  (common service)
# thanos-receive-shard-0          ClusterIP   ...
# thanos-receive-shard-0-headless ClusterIP   None ...
# thanos-receive-shard-1          ClusterIP   ...
# thanos-receive-shard-1-headless ClusterIP   None ...
```

Verify hashring configuration:

```bash
kubectl get configmap thanos-receive-hashrings -n monitoring -o yaml
```

You should see all endpoints from all shards listed in the hashring.

## How It Works

### Hashring Distribution

The hashring automatically includes all replicas from all shards. For example, with 2 shards and 3 replicas each:

```json
[
  {
    "endpoints": [
      "thanos-receive-shard-0-0.thanos-receive-shard-0-headless.default.svc.cluster.local:10901",
      "thanos-receive-shard-0-1.thanos-receive-shard-0-headless.default.svc.cluster.local:10901",
      "thanos-receive-shard-0-2.thanos-receive-shard-0-headless.default.svc.cluster.local:10901",
      "thanos-receive-shard-1-0.thanos-receive-shard-1-headless.default.svc.cluster.local:10901",
      "thanos-receive-shard-1-1.thanos-receive-shard-1-headless.default.svc.cluster.local:10901",
      "thanos-receive-shard-1-2.thanos-receive-shard-1-headless.default.svc.cluster.local:10901"
    ]
  }
]
```

### Remote Write Target

Point your Prometheus instances to the common service:

```yaml
remote_write:
  - url: http://thanos-receive.monitoring.svc.cluster.local:10908/api/v1/receive
```

The common service load-balances across all shards, and the hashring ensures consistent routing.

### Labels

Each shard pod gets automatic labels:
- `app.kubernetes.io/component: receive`
- `app.kubernetes.io/shard: <shard-name>`
- `receive_shard: <shard-name>` (as a Prometheus label)
- `receive_replica: <pod-name>` (as a Prometheus label)

## Migration from Single-Shard

To migrate from single-shard to multi-shard:

1. **Deploy new shards** alongside existing receive
2. **Update Prometheus** to write to the new common service
3. **Wait for old data** to age out or copy to new buckets
4. **Remove old deployment**

Alternatively, you can keep the legacy mode and gradually migrate.

## Scaling

### Adding a New Shard

Simply add a new entry to `receive.shards`:

```yaml
receive:
  shards:
    - name: shard-0
      # ...existing config
    - name: shard-1
      # ...existing config
    - name: shard-3  # New shard
      replicaCount: 3
      objstore:
        secretName: thanos-receive-shard-3-objstore
        secretKey: objstore.yml
```

Don't forget to create the secret first!

### Scaling Replicas within a Shard

Modify `replicaCount` for the specific shard:

```yaml
receive:
  shards:
    - name: shard-0
      replicaCount: 5  # Increased from 3
```

## Troubleshooting

### Check Hashring Configuration

```bash
kubectl exec -it thanos-receive-shard-0-0 -n monitoring -- cat /etc/thanos/hashrings.json
```

### Verify Object Storage Secrets

```bash
kubectl get secret thanos-receive-shard-0-objstore -n monitoring -o yaml
```

### Check Shard Labels

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/component=receive --show-labels
```

### View Logs

```bash
kubectl logs -n monitoring thanos-receive-shard-0-0 -f
```

## Best Practices

1. **One bucket per shard**: Always use separate S3 buckets for each shard
2. **Consistent sizing**: Start with equal replica counts across shards for balanced load
3. **Monitor metrics**: Track `thanos_receive_forward_*` metrics to ensure proper distribution
4. **Plan capacity**: Size persistence volumes appropriately based on write rate and retention
5. **Test failover**: Verify behavior when a shard is down
6. **Use PDBs**: Enable PodDisruptionBudgets to prevent too many simultaneous disruptions

## See Also

- [Thanos Receive Documentation](https://thanos.io/tip/components/receive.md/)
- [Example values file](./examples/receive-multi-shard-values.yaml)
- [Thanos Hashring Configuration](https://thanos.io/tip/components/receive.md/#hashring-configuration)
