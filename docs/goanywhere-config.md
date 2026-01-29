# GoAnywhere MFT Configuration Guide

This document describes how to configure GoAnywhere MFT (running on Windows) to send files to the AMQ Broker on OpenShift.

## Overview

GoAnywhere monitors local directories for new files and sends them to the AMQ Broker queue when detected. The Camel integration on OpenShift then processes these files through the document processing pipeline.

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           Windows Server                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                     GoAnywhere MFT                                   │ │
│  │  ┌───────────────┐    ┌──────────────┐    ┌────────────────────┐   │ │
│  │  │ File Monitor  │───▶│   Workflow   │───▶│   JMS Connector    │   │ │
│  │  │  (Trigger)    │    │  (Process)   │    │  (AMQP/OpenWire)   │   │ │
│  │  └───────────────┘    └──────────────┘    └─────────┬──────────┘   │ │
│  └─────────────────────────────────────────────────────┼───────────────┘ │
└────────────────────────────────────────────────────────┼─────────────────┘
                                                         │
                                            AMQPS (Port 443)
                                                         │
                                                         ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                         OpenShift Cluster                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      file-pipeline namespace                          │  │
│  │  ┌────────────────────────────────────────┐                          │  │
│  │  │           AMQ Broker Route             │◀─────────────────────────│──│
│  │  │  (file-pipeline-broker-amqp-0-svc-rte) │                          │  │
│  │  └────────────────────────────────────────┘                          │  │
│  │                         │                                             │  │
│  │                         ▼                                             │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │  │                 Camel Integration                                │ │  │
│  │  └─────────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

1. GoAnywhere MFT 7.x or later installed on Windows Server
2. Java Runtime Environment (JRE) 11 or later
3. Network connectivity from Windows server to OpenShift cluster (HTTPS/443)
4. AMQ Broker deployed and accessible via OpenShift Route

## Step 1: Obtain Connection Details

### Get the AMQ Broker Route URL

```bash
# From OpenShift CLI
oc get route -n file-pipeline

# Example output:
# NAME                                    HOST/PORT
# file-pipeline-broker-amqp-0-svc-rte    file-pipeline-broker-amqp-0-svc-rte-file-pipeline.apps.cluster.example.com
```

### Get Broker Credentials

The broker admin credentials are configured in the `amq-credentials` secret. For production, create a dedicated service account.

## Step 2: Configure JMS Resource in GoAnywhere

### Create JMS Connection Factory

1. Navigate to **System** → **Resources** → **JMS**
2. Click **Add JMS Connection**
3. Configure the following:

| Field | Value |
|-------|-------|
| **Name** | `AMQ-Broker-OpenShift` |
| **Connection Factory Class** | `org.apache.qpid.jms.JmsConnectionFactory` |
| **Provider URL** | `amqps://file-pipeline-broker-amqp-0-svc-rte-file-pipeline.apps.<cluster-domain>:443` |
| **Username** | `camel-user` (from secret) |
| **Password** | (from secret) |

### JNDI Properties

Add the following JNDI properties:

```properties
# Connection factory settings
connectionFactory.ConnectionFactory=amqps://file-pipeline-broker-amqp-0-svc-rte-file-pipeline.apps.<cluster-domain>:443

# Queue definition
queue.file-transfer-queue=file-transfer-queue

# SSL/TLS settings (if using custom CA)
transport.trustStoreLocation=C:/GoAnywhere/security/truststore.jks
transport.trustStorePassword=changeit
transport.verifyHost=true
```

### Required JAR Files

Download and add these JARs to GoAnywhere's classpath (`<GoAnywhere>/lib/`):

1. **Qpid JMS Client**: `qpid-jms-client-2.x.x.jar`
2. **Proton-J**: `proton-j-0.34.x.jar`
3. **Netty**: Required Netty libraries for AMQP transport

Alternative: Use **ActiveMQ Client** for OpenWire protocol:
- `activemq-client-5.x.x.jar` and dependencies

## Step 3: Create File Monitor (Trigger)

### Configure Directory Monitor

1. Navigate to **Workflows** → **Triggers**
2. Click **Add Trigger** → **File Monitor**
3. Configure:

| Field | Value |
|-------|-------|
| **Name** | `IncomingDocuments-Monitor` |
| **Monitor Directory** | `C:\Documents\Incoming` |
| **File Pattern** | `*.*` or specific patterns like `*.pdf;*.docx` |
| **Trigger Type** | On File Arrival |
| **Minimum Age** | 5 seconds (ensures file is fully written) |
| **Check Interval** | 30 seconds |

### File Stability Check

Enable file stability to ensure files are completely written:

```
☑ Check file stability
   Stable Time: 5 seconds
   Check Method: Size check
```

## Step 4: Create Processing Workflow

### Workflow Overview

1. **Read File** - Read the incoming file
2. **Calculate Checksum** - Generate SHA-256 hash
3. **Set Variables** - Prepare message headers
4. **Send to JMS** - Send to AMQ Broker queue
5. **Move/Archive** - Move processed file to archive

### Workflow Steps

#### Step 1: Read File

```
Task: Read File
  Source: ${trigger.filePath}
  Read as Binary: Yes
```

#### Step 2: Calculate Checksum

```
Task: Calculate Checksum
  Algorithm: SHA-256
  Input: ${file.content}
  Output Variable: fileChecksum
```

#### Step 3: Set Variables

```
Task: Set Variables
  correlationId = ${uuid()}
  transferId = GOANYWHERE-${timestamp(yyyyMMddHHmmssSSS)}
  originalFileName = ${trigger.fileName}
  contentType = ${mimeType(trigger.fileName)}
  fileSize = ${trigger.fileSize}
```

#### Step 4: Send to JMS Queue

```
Task: Write to JMS Queue
  Connection: AMQ-Broker-OpenShift
  Destination Type: Queue
  Queue Name: file-transfer-queue

  Message Properties:
    fileName: ${originalFileName}
    contentType: ${contentType}
    fileSize: ${fileSize}
    transferId: ${transferId}
    checksum: ${fileChecksum}

  JMS Headers:
    JMSCorrelationID: ${correlationId}
    JMSType: FileTransfer

  Message Body: ${file.content}
  Body Type: Bytes
```

#### Step 5: Archive Processed File

```
Task: Move File
  Source: ${trigger.filePath}
  Destination: C:\Documents\Archive\${date(yyyy-MM-dd)}\${trigger.fileName}
  Create Directories: Yes
  Overwrite: Rename (add timestamp)
```

### Error Handling

Configure error handling in the workflow:

```
On Error:
  ☑ Retry enabled
     Max Retries: 3
     Retry Delay: 30 seconds
     Backoff Multiplier: 2

  On Final Failure:
     Move to: C:\Documents\Failed\
     Send Alert: Yes
     Alert Email: ops-team@example.com
```

## Step 5: SSL/TLS Configuration

### Import OpenShift CA Certificate

1. Export the OpenShift cluster CA certificate:

```bash
oc get secret router-ca -n openshift-ingress-operator -o jsonpath='{.data.tls\.crt}' | base64 -d > openshift-ca.crt
```

2. Import into GoAnywhere truststore:

```cmd
keytool -import -alias openshift-ca ^
  -file openshift-ca.crt ^
  -keystore C:\GoAnywhere\security\truststore.jks ^
  -storepass changeit
```

3. Configure GoAnywhere to use the truststore:
   - **System** → **Global Settings** → **SSL/TLS**
   - Set Trust Store path and password

## Step 6: Testing

### Test JMS Connection

1. Navigate to **Resources** → **JMS**
2. Select `AMQ-Broker-OpenShift`
3. Click **Test Connection**
4. Verify: "Connection successful"

### Send Test Message

1. Create a test file in the monitored directory:

```cmd
echo "Test document content" > C:\Documents\Incoming\test.txt
```

2. Monitor the workflow execution in GoAnywhere logs
3. Verify message arrival in AMQ Broker console:
   - Access: `https://file-pipeline-broker-wconsj-0-svc-rte-file-pipeline.apps.<cluster-domain>`
   - Navigate to **Queues** → **file-transfer-queue**
   - Verify message count incremented

### Verify End-to-End

1. Check AMQ Broker queue statistics
2. Verify file in S3 `incoming/` folder
3. Verify processed result in S3 `processed/` folder

## Monitoring

### GoAnywhere Logs

- **Location**: `<GoAnywhere>/userdata/logs/`
- **Workflow Logs**: Check for JMS connection and message delivery
- **Error Logs**: `error.log` for connection failures

### Recommended Log Settings

```
Workflow Logging: Detailed
JMS Debug: Enabled (for troubleshooting)
Retention: 30 days
```

## Troubleshooting

### Connection Refused

```
Error: Unable to connect to AMQ Broker
```

**Solutions**:
1. Verify network connectivity: `telnet <route-host> 443`
2. Check firewall rules allow outbound HTTPS
3. Verify Route is exposed: `oc get route -n file-pipeline`

### SSL Handshake Failed

```
Error: SSL handshake exception
```

**Solutions**:
1. Verify CA certificate is imported correctly
2. Check certificate expiration
3. Ensure TLS 1.2+ is enabled

### Authentication Failed

```
Error: Security exception - Invalid credentials
```

**Solutions**:
1. Verify username/password match AMQ Broker configuration
2. Check if user has permission to send to queue
3. Review AMQ Broker security settings

### Message Not Delivered

```
Error: Message send timeout
```

**Solutions**:
1. Check AMQ Broker is running: `oc get pods -n file-pipeline`
2. Verify queue exists: Check AMQ Console
3. Review message size limits

## Security Best Practices

1. **Use Dedicated Service Account**: Don't use admin credentials
2. **Enable TLS**: Always use `amqps://` (TLS-encrypted AMQP)
3. **Rotate Credentials**: Implement regular credential rotation
4. **Audit Logging**: Enable detailed logging for compliance
5. **Network Segmentation**: Restrict outbound access to known endpoints
6. **File Validation**: Validate file types before processing

## Reference

### Message Header Format

| Header | Type | Required | Description |
|--------|------|----------|-------------|
| `fileName` | String | Yes | Original filename with extension |
| `contentType` | String | Yes | MIME type (e.g., `application/pdf`) |
| `fileSize` | Long | Yes | File size in bytes |
| `transferId` | String | Yes | Unique transfer identifier |
| `checksum` | String | Yes | SHA-256 hash of file content |
| `JMSCorrelationID` | String | Yes | UUID for correlation |

### Supported File Types

- PDF documents (`.pdf`)
- Microsoft Office (`.docx`, `.xlsx`, `.pptx`)
- Images (`.png`, `.jpg`, `.tiff`)
- Plain text (`.txt`, `.csv`)

Maximum file size: 100 MB (configurable in AMQ Broker)
