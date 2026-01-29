# File Transfer Pipeline

A file transfer and document processing pipeline running on OpenShift, integrating GoAnywhere MFT, Red Hat AMQ Broker, Apache Camel, Docling, and OpenShift Data Foundation.

## Architecture

```
┌──────────────┐                    ┌───────────────────────────────────────────────┐
│  GoAnywhere  │   AMQP/OpenWire    │              OpenShift Cluster                │
│  (Windows)   │────────────────────┼──┐                                            │
└──────────────┘                    │  │  ┌───────────────────────────────────────┐ │
                                    │  │  │        file-pipeline namespace        │ │
                                    │  │  │                                       │ │
                                    │  │  │  ┌─────────────┐                      │ │
                                    │  └──┼─▶│ AMQ Broker  │◀─┐                   │ │
                                    │     │  │ (Artemis)   │  │                   │ │
                                    │     │  └──────┬──────┘  │                   │ │
                                    │     │         │         │                   │ │
                                    │     │         ▼         │                   │ │
                                    │     │  ┌─────────────┐  │   ┌────────────┐  │ │
                                    │     │  │   Camel     │──┼──▶│  ODF S3    │  │ │
                                    │     │  │   Quarkus   │  │   │  Bucket    │  │ │
                                    │     │  └──────┬──────┘  │   └────────────┘  │ │
                                    │     │         │         │                   │ │
                                    │     │         ▼         │                   │ │
                                    │     │  ┌─────────────┐  │                   │ │
                                    │     │  │   Docling   │  │                   │ │
                                    │     │  │   Service   │  │                   │ │
                                    │     │  └─────────────┘  │                   │ │
                                    │     │                                       │ │
                                    │     └───────────────────────────────────────┘ │
                                    └───────────────────────────────────────────────┘
```

## Components

| Component | Description | Technology |
|-----------|-------------|------------|
| **GoAnywhere MFT** | File monitoring and transfer initiation | Windows, JMS/AMQP |
| **AMQ Broker** | Message queue for reliable file transfer | Red Hat AMQ 7.12 (Artemis) |
| **Camel Quarkus** | Integration pipeline orchestration | Apache Camel 4.x, Quarkus 3.x |
| **Docling** | Document processing (PDF, Office, images) | Python, AI/ML models |
| **ODF S3** | Object storage for files | OpenShift Data Foundation |

## Pipeline Flow

1. **File Detection**: GoAnywhere monitors directories for new files
2. **Message Queue**: File content sent to AMQ Broker with metadata headers
3. **S3 Upload**: Camel stores raw file in S3 `incoming/` folder
4. **Document Processing**: Docling extracts text, tables, and structure
5. **Result Storage**: Processed JSON stored in S3 `processed/` folder
6. **Error Handling**: Failed messages go to DLQ with retry logic

## Project Structure

```
fbg-kafka-stream-file-transfer/
├── devfile.yaml                    # OpenShift DevSpaces configuration
├── k8s/
│   ├── namespace.yaml              # Namespace + resource quotas
│   ├── object-bucket-claim.yaml    # ODF S3 bucket
│   ├── amq-broker.yaml             # AMQ Broker instance
│   ├── amq-address.yaml            # Queue definitions
│   ├── configmaps.yaml             # Application configuration
│   ├── secrets.yaml                # Credentials (templates)
│   ├── docling-deployment.yaml     # Docling service
│   ├── docling-service.yaml        # Docling ClusterIP
│   ├── camel-deployment.yaml       # Camel integration
│   ├── camel-service.yaml          # Camel ClusterIP
│   └── monitoring/
│       ├── servicemonitors.yaml    # Prometheus scraping
│       └── alerts.yaml             # Alert rules
├── camel-integration/
│   ├── pom.xml                     # Maven project
│   ├── src/main/resources/
│   │   ├── application.properties  # App configuration
│   │   └── camel/
│   │       ├── file-pipeline.yaml  # Main processing route
│   │       └── dlq-handler.yaml    # Dead letter queue handler
│   └── Dockerfile                  # Container build
└── docs/
    ├── goanywhere-config.md        # GoAnywhere setup guide
    └── deployment.md               # Deployment procedures
```

## Quick Start

### Prerequisites

- OpenShift 4.14+
- Red Hat AMQ Broker Operator
- OpenShift Data Foundation
- `oc` CLI authenticated to cluster

### Deploy

```bash
# Clone repository
git clone <repository-url>
cd fbg-kafka-stream-file-transfer

# Create namespace
oc apply -f k8s/namespace.yaml

# Deploy components (in order)
oc apply -f k8s/object-bucket-claim.yaml
oc apply -f k8s/amq-broker.yaml
oc apply -f k8s/amq-address.yaml
oc apply -f k8s/secrets.yaml
oc apply -f k8s/configmaps.yaml
oc apply -f k8s/docling-deployment.yaml
oc apply -f k8s/docling-service.yaml

# Build and deploy Camel integration
oc new-build --name=camel-integration --binary --strategy=docker -n file-pipeline
oc start-build camel-integration --from-dir=camel-integration -n file-pipeline --follow
oc apply -f k8s/camel-deployment.yaml
oc apply -f k8s/camel-service.yaml
```

### Verify

```bash
# Check all pods are running
oc get pods -n file-pipeline

# Check routes (for GoAnywhere connection)
oc get routes -n file-pipeline
```

## Development

### Using OpenShift DevSpaces

1. Open the repository in DevSpaces
2. Run `dev-mode` command for Quarkus live coding
3. Use Kaoto extension to visually edit Camel routes

### Local Development

```bash
cd camel-integration

# Start with dev services (Artemis, MinIO)
mvn quarkus:dev

# Run tests
mvn test
```

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `QUARKUS_ARTEMIS_URL` | AMQ Broker connection URL | `tcp://localhost:61616` |
| `S3_BUCKET_NAME` | S3 bucket name | `file-pipeline` |
| `DOCLING_SERVICE_URL` | Docling service URL | `http://docling-service:5001` |

### Message Headers (GoAnywhere → AMQ)

| Header | Required | Description |
|--------|----------|-------------|
| `fileName` | Yes | Original filename |
| `contentType` | Yes | MIME type |
| `fileSize` | Yes | Size in bytes |
| `transferId` | Yes | Unique transfer ID |
| `checksum` | Yes | SHA-256 hash |

## Monitoring

- **Prometheus metrics**: `/q/metrics` endpoint
- **Health checks**: `/q/health/live`, `/q/health/ready`
- **AMQ Console**: Web UI for queue management
- **Alerts**: Pre-configured PrometheusRules

## Documentation

- [Deployment Guide](docs/deployment.md) - Full deployment procedures
- [GoAnywhere Configuration](docs/goanywhere-config.md) - Windows setup guide

## Technology Stack

- **Red Hat AMQ Broker 7.12** - Message queuing
- **Red Hat build of Apache Camel 4.x** - Integration framework
- **Quarkus 3.x** - Container-optimized Java runtime
- **Docling Serve** - Document AI processing
- **OpenShift Data Foundation** - S3 storage
- **OpenShift 4.14+** - Container platform

## License

[Add license information]
