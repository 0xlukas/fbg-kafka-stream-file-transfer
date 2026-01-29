# Deployment Guide

This guide covers deploying the File Transfer Pipeline to OpenShift.

## Prerequisites

### Cluster Requirements

- OpenShift 4.14 or later
- Cluster admin access (for operator installation)
- At least 16 GB RAM available for the pipeline components

### Required Operators

Install these operators from OperatorHub before deployment:

1. **Red Hat AMQ Broker** (v7.12+)
   - Provides ActiveMQ Artemis for message queuing

2. **OpenShift Data Foundation** (ODF)
   - Provides S3-compatible object storage via NooBaa

3. **Red Hat OpenShift Dev Spaces** (optional)
   - Cloud-based IDE for development

### Verify Operators

```bash
# Check installed operators
oc get csv -n openshift-operators | grep -E "amq|ocs|devspaces"
```

## Deployment Steps

### Step 1: Create Namespace

```bash
cd /path/to/fbg-kafka-stream-file-transfer

# Create namespace and resource quotas
oc apply -f k8s/namespace.yaml

# Verify
oc get namespace file-pipeline
oc get resourcequota -n file-pipeline
```

### Step 2: Create S3 Bucket

```bash
# Create ObjectBucketClaim for ODF S3 storage
oc apply -f k8s/object-bucket-claim.yaml

# Wait for bucket provisioning (may take 1-2 minutes)
oc get objectbucketclaim file-pipeline-bucket -n file-pipeline -w

# Verify ConfigMap and Secret were created
oc get configmap file-pipeline-bucket -n file-pipeline
oc get secret file-pipeline-bucket -n file-pipeline
```

### Step 3: Deploy AMQ Broker

```bash
# Create AMQ Broker instance
oc apply -f k8s/amq-broker.yaml

# Wait for broker pods to be ready
oc get pods -n file-pipeline -l ActiveMQArtemis=file-pipeline-broker -w

# Create queue addresses
oc apply -f k8s/amq-address.yaml

# Verify queues
oc get activemqartemisaddress -n file-pipeline
```

### Step 4: Create TLS Certificate (Production)

For production, replace the placeholder TLS certificate:

```bash
# Option 1: Use OpenShift service serving certificates
oc annotate service file-pipeline-broker-amqp-0-svc \
  service.beta.openshift.io/serving-cert-secret-name=amq-broker-tls \
  -n file-pipeline

# Option 2: Use cert-manager or external certificate
# Replace the secret with your actual certificates
oc create secret tls amq-broker-tls \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key \
  -n file-pipeline \
  --dry-run=client -o yaml | oc apply -f -
```

### Step 5: Create Secrets and ConfigMaps

```bash
# Create application secrets (edit values first!)
# IMPORTANT: Change default passwords before applying
vim k8s/secrets.yaml

oc apply -f k8s/secrets.yaml
oc apply -f k8s/configmaps.yaml
```

### Step 6: Deploy Docling Service

```bash
# Deploy Docling
oc apply -f k8s/docling-deployment.yaml
oc apply -f k8s/docling-service.yaml

# Wait for Docling pods (may take several minutes for model download)
oc get pods -n file-pipeline -l app.kubernetes.io/name=docling -w

# Check Docling logs
oc logs -f deployment/docling-service -n file-pipeline
```

### Step 7: Build and Deploy Camel Integration

#### Option A: Build in OpenShift (Recommended)

```bash
# Create BuildConfig
oc new-build --name=camel-integration \
  --binary \
  --strategy=docker \
  -n file-pipeline

# Start build
oc start-build camel-integration \
  --from-dir=camel-integration \
  -n file-pipeline \
  --follow

# Deploy
oc apply -f k8s/camel-deployment.yaml
oc apply -f k8s/camel-service.yaml
```

#### Option B: Build Locally and Push

```bash
# Build locally
cd camel-integration
mvn package -DskipTests

# Build container
docker build -t camel-integration:latest .

# Tag and push to OpenShift registry
oc registry login
docker tag camel-integration:latest \
  image-registry.openshift-image-registry.svc:5000/file-pipeline/camel-integration:latest
docker push image-registry.openshift-image-registry.svc:5000/file-pipeline/camel-integration:latest

# Deploy
oc apply -f k8s/camel-deployment.yaml
oc apply -f k8s/camel-service.yaml
```

### Step 8: Configure Monitoring (Optional)

```bash
# Deploy ServiceMonitors and alerts
oc apply -f k8s/monitoring/servicemonitors.yaml
oc apply -f k8s/monitoring/alerts.yaml

# Verify ServiceMonitors
oc get servicemonitor -n file-pipeline
```

## Verification

### Check All Pods Running

```bash
oc get pods -n file-pipeline

# Expected output:
# NAME                                  READY   STATUS    RESTARTS   AGE
# camel-integration-xxx                 1/1     Running   0          5m
# docling-service-xxx                   1/1     Running   0          10m
# file-pipeline-broker-ss-0             1/1     Running   0          15m
# file-pipeline-broker-ss-1             1/1     Running   0          14m
```

### Check Services

```bash
oc get svc -n file-pipeline

# Expected services:
# - camel-integration
# - docling-service
# - file-pipeline-broker-*-svc
```

### Check Routes

```bash
oc get routes -n file-pipeline

# External routes for:
# - AMQ Broker AMQP (for GoAnywhere)
# - AMQ Console (for management)
```

### Test AMQ Broker Connection

```bash
# Port-forward to broker
oc port-forward svc/file-pipeline-broker-all-0-svc 61618:61618 -n file-pipeline

# Send test message (requires artemis CLI)
# Or use AMQ Console web interface
```

### Test Docling Service

```bash
# Port-forward to Docling
oc port-forward svc/docling-service 5001:5001 -n file-pipeline

# Test health endpoint
curl http://localhost:5001/health

# Test conversion (with a sample file)
curl -X POST http://localhost:5001/v1/convert/source \
  -H "Content-Type: application/json" \
  -d '{"source": "https://example.com/sample.pdf"}'
```

### Test Camel Integration

```bash
# Check health
oc port-forward svc/camel-integration 9000:9000 -n file-pipeline
curl http://localhost:9000/q/health

# Check metrics
curl http://localhost:9000/q/metrics
```

## End-to-End Test

1. **Configure GoAnywhere** following `docs/goanywhere-config.md`

2. **Place test file** in GoAnywhere monitored directory

3. **Monitor AMQ Broker Console**:
   - Access: `https://<amq-console-route>`
   - Check `file-transfer-queue` message count

4. **Check S3 buckets**:
   ```bash
   # Get NooBaa CLI or use S3 browser
   # Check incoming/ folder for raw file
   # Check processed/ folder for Docling output
   ```

5. **Monitor Camel logs**:
   ```bash
   oc logs -f deployment/camel-integration -n file-pipeline
   ```

## Scaling

### Manual Scaling

```bash
# Scale Camel integration
oc scale deployment camel-integration --replicas=4 -n file-pipeline

# Scale Docling (if GPU available, keep lower)
oc scale deployment docling-service --replicas=3 -n file-pipeline
```

### Autoscaling

HorizontalPodAutoscalers are included in the deployments. Verify they're working:

```bash
oc get hpa -n file-pipeline
```

## Troubleshooting

### Camel Integration Not Connecting to AMQ

```bash
# Check Camel logs
oc logs deployment/camel-integration -n file-pipeline | grep -i artemis

# Verify AMQ service is reachable
oc run debug --rm -it --image=busybox -n file-pipeline -- \
  nc -zv file-pipeline-broker-all-0-svc 61618
```

### Docling Processing Slow

```bash
# Check Docling resource usage
oc top pods -n file-pipeline -l app.kubernetes.io/name=docling

# Check if model cache is populated
oc exec deployment/docling-service -n file-pipeline -- ls -la /models
```

### S3 Connection Issues

```bash
# Verify bucket credentials
oc get secret file-pipeline-bucket -n file-pipeline -o jsonpath='{.data}' | base64 -d

# Test S3 connection from Camel pod
oc exec deployment/camel-integration -n file-pipeline -- \
  curl -v https://s3.openshift-storage.svc
```

### Messages Going to DLQ

```bash
# Check DLQ message count
# Via AMQ Console or:
oc exec file-pipeline-broker-ss-0 -n file-pipeline -- \
  /opt/amq/bin/artemis queue stat --url tcp://localhost:61617

# Check failure reports in S3
# Look in failed/ folder
```

## Rollback

### Rollback Deployment

```bash
# Rollback to previous revision
oc rollout undo deployment/camel-integration -n file-pipeline

# Check rollout history
oc rollout history deployment/camel-integration -n file-pipeline
```

### Restore Previous ConfigMap

```bash
# Get previous ConfigMap from backup or git
oc apply -f backup/configmaps.yaml
oc rollout restart deployment/camel-integration -n file-pipeline
```

## Cleanup

### Remove All Resources

```bash
# Delete all resources in namespace
oc delete namespace file-pipeline

# Or delete individually
oc delete -f k8s/camel-deployment.yaml
oc delete -f k8s/docling-deployment.yaml
oc delete -f k8s/amq-broker.yaml
oc delete -f k8s/object-bucket-claim.yaml
```

## Production Checklist

Before going to production:

- [ ] Replace all default passwords in secrets
- [ ] Configure proper TLS certificates
- [ ] Set appropriate resource limits
- [ ] Enable and verify monitoring/alerting
- [ ] Configure backup for AMQ Broker PVCs
- [ ] Document runbooks for common issues
- [ ] Set up log aggregation (EFK/Loki)
- [ ] Configure network policies for isolation
- [ ] Enable pod security policies/standards
- [ ] Test disaster recovery procedures
