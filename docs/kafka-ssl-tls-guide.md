# Kafka SSL/TLS Authentication Guide

This guide explains how to configure HolmesGPT to connect to Kafka clusters using SSL/TLS (mutual TLS) authentication.

## Overview

The Kafka toolset now supports SSL/TLS authentication with the following features:

- **Mutual TLS (mTLS)**: Kafka brokers verify the client's certificate, and clients verify the broker's certificate
- **Flexible Certificate Handling**: Support for both mounted certificates and base64-encoded credentials
- **PEM Format Support**: All certificates must be in PEM format (no JKS conversion needed)
- **Coexistence with SASL**: Can use SSL/TLS alongside SASL authentication (SASL_SSL)

## Prerequisites

Before configuring Kafka SSL/TLS, you need:

1. **CA Certificate**: The certificate authority certificate for verifying the Kafka broker
2. **Client Certificate**: The client certificate for mTLS authentication
3. **Client Private Key**: The private key corresponding to the client certificate
4. **All in PEM format**: Certificates must be in PEM format (-----BEGIN CERTIFICATE-----)

## Configuration Methods

### Method 1: Mounted Certificates (Recommended for Production)

This method mounts certificates from Kubernetes secrets, providing better security.

#### Step 1: Create Kubernetes Secrets

```bash
# Create secret for CA certificate
kubectl create secret generic kafka-ca-cert \
  --from-file=ca.crt=path/to/ca.crt \
  -n <namespace>

# Create secret for client certificate and key
kubectl create secret tls kafka-client-certs \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key \
  -n <namespace>
```

#### Step 2: Update Helm values.yaml

```yaml
kafkaSSL:
  enabled: true
  useMountedCertificates: true
  mountPath: "/etc/kafka/ssl"
  caSecret:
    name: "kafka-ca-cert"
    key: "ca.crt"
  clientSecret:
    name: "kafka-client-certs"
    certKey: "tls.crt"
    keyKey: "tls.key"
```

#### Step 3: Configure Kafka Cluster

In your toolset configuration:

```yaml
kafka:
  enabled: true
  clusters:
    - name: "my-kafka-cluster"
      broker: "kafka-broker.example.com:9093"
      security_protocol: "SSL"
      ssl_ca_cert_path: "/etc/kafka/ssl/ca.crt"
      ssl_client_cert_path: "/etc/kafka/ssl/tls.crt"
      ssl_client_key_path: "/etc/kafka/ssl/tls.key"
```

### Method 2: Base64-Encoded Certificates

This method embeds base64-encoded certificates in the ConfigMap, simpler but less secure.

#### Step 1: Encode Certificates

```bash
# Encode certificates to base64
cat ca.crt | base64 -w 0 > ca.crt.b64
cat tls.crt | base64 -w 0 > tls.crt.b64
cat tls.key | base64 -w 0 > tls.key.b64
```

#### Step 2: Create Kubernetes Secrets with Base64 Content

```bash
kubectl create secret generic kafka-ca-cert \
  --from-literal=ca.crt="$(cat ca.crt.b64)" \
  -n <namespace>

kubectl create secret generic kafka-client-certs \
  --from-literal=tls.crt="$(cat tls.crt.b64)" \
  --from-literal=tls.key="$(cat tls.key.b64)" \
  -n <namespace>
```

#### Step 3: Update Helm values.yaml

```yaml
kafkaSSL:
  enabled: true
  useMountedCertificates: false
  caSecret:
    name: "kafka-ca-cert"
    key: "ca.crt"
  clientSecret:
    name: "kafka-client-certs"
    certKey: "tls.crt"
    keyKey: "tls.key"

additionalEnvVars:
  - name: KAFKA_CA_CERT_BASE64
    valueFrom:
      secretKeyRef:
        name: kafka-ca-cert
        key: ca.crt
  - name: KAFKA_CLIENT_CERT_BASE64
    valueFrom:
      secretKeyRef:
        name: kafka-client-certs
        key: tls.crt
  - name: KAFKA_CLIENT_KEY_BASE64
    valueFrom:
      secretKeyRef:
        name: kafka-client-certs
        key: tls.key
```

#### Step 4: Configure Kafka Cluster

```yaml
kafka:
  enabled: true
  clusters:
    - name: "my-kafka-cluster"
      broker: "kafka-broker.example.com:9093"
      security_protocol: "SSL"
      ssl_ca_cert: "{{ env.KAFKA_CA_CERT_BASE64 }}"
      ssl_client_cert: "{{ env.KAFKA_CLIENT_CERT_BASE64 }}"
      ssl_client_key: "{{ env.KAFKA_CLIENT_KEY_BASE64 }}"
```

## Combined SSL + SASL Authentication

You can combine SSL/TLS with SASL authentication for additional security:

```yaml
kafka:
  enabled: true
  clusters:
    - name: "secure-kafka"
      broker: "kafka.example.com:9093"
      security_protocol: "SASL_SSL"
      sasl_mechanism: "PLAIN"
      username: "{{ env.KAFKA_USERNAME }}"
      password: "{{ env.KAFKA_PASSWORD }}"
      ssl_ca_cert_path: "/etc/kafka/ssl/ca.crt"
      ssl_client_cert_path: "/etc/kafka/ssl/tls.crt"
      ssl_client_key_path: "/etc/kafka/ssl/tls.key"
```

## Configuration Reference

### KafkaClusterConfig SSL Fields

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `ssl_ca_cert` | string | Base64-encoded CA certificate (PEM format) | `{{ env.KAFKA_CA_CERT_BASE64 }}` |
| `ssl_client_cert` | string | Base64-encoded client certificate (PEM format) | `{{ env.KAFKA_CLIENT_CERT_BASE64 }}` |
| `ssl_client_key` | string | Base64-encoded client private key (PEM format) | `{{ env.KAFKA_CLIENT_KEY_BASE64 }}` |
| `ssl_ca_cert_path` | string | Path to mounted CA certificate file | `/etc/kafka/ssl/ca.crt` |
| `ssl_client_cert_path` | string | Path to mounted client certificate file | `/etc/kafka/ssl/tls.crt` |
| `ssl_client_key_path` | string | Path to mounted client private key file | `/etc/kafka/ssl/tls.key` |
| `security_protocol` | string | Security protocol to use | `SSL` or `SASL_SSL` |

### Helm Values Configuration

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `kafkaSSL.enabled` | boolean | `false` | Enable Kafka SSL/TLS configuration |
| `kafkaSSL.useMountedCertificates` | boolean | `true` | Use mounted certificates (vs. base64) |
| `kafkaSSL.mountPath` | string | `/etc/kafka/ssl` | Base path for mounted certificates |
| `kafkaSSL.caSecret.name` | string | `""` | Kubernetes secret name for CA certificate |
| `kafkaSSL.caSecret.key` | string | `ca.crt` | Key in secret for CA certificate |
| `kafkaSSL.clientSecret.name` | string | `""` | Kubernetes secret name for client cert/key |
| `kafkaSSL.clientSecret.certKey` | string | `tls.crt` | Key in secret for client certificate |
| `kafkaSSL.clientSecret.keyKey` | string | `tls.key` | Key in secret for client private key |

## Troubleshooting

### Certificate Validation Errors

**Error**: `SSL: CERTIFICATE_VERIFY_FAILED`

**Solution**: Ensure the CA certificate is correct and matches the Kafka broker's certificate chain.

### File Permission Errors

**Error**: `Permission denied` when reading certificate files

**Solution**: The Helm template automatically sets appropriate permissions. If using base64 method, ensure the temporary files are readable.

### Connection Timeout

**Error**: Connection times out when connecting to Kafka

**Solution**:

- Verify the broker address and port are correct
- Ensure the client certificate is valid for the broker's hostname
- Check network connectivity to the Kafka broker

### Certificate Decode Errors

**Error**: `Failed to decode CA certificate` or similar

**Solution**:

- Ensure certificates are properly base64-encoded (use `base64 -w 0` to avoid line breaks)
- Verify the certificate content is valid PEM format
- Check that environment variables are correctly set

## Security Best Practices

1. **Use Mounted Certificates**: Prefer mounted certificates over base64 encoding for production
2. **Restrict Secret Access**: Limit Kubernetes secret access to necessary service accounts
3. **Read-Only Mounts**: Certificates are mounted as read-only for security
4. **Key Permissions**: Private keys are automatically restricted to 0600 permissions
5. **Temporary Files**: Base64-encoded certificates are written to temporary files and cleaned up after client initialization

## Examples

See `/examples/kafka-ssl-config.yaml` for complete configuration examples including:
- Mounted certificate configuration
- Base64-encoded certificate configuration
- Combined SSL + SASL configuration

## Implementation Details

### How It Works

1. **Configuration Loading**: When HolmesGPT starts, it loads the Kafka configuration from the ConfigMap
2. **Certificate Handling**: 
   - For mounted paths: Uses the provided file paths directly
   - For base64 content: Decodes and writes to temporary files
3. **Client Setup**: Configures the Confluent Kafka AdminClient with SSL settings
4. **Connection Test**: Tests the connection by listing topics with a 10-second timeout
5. **Cleanup**: Temporary files are cleaned up after client initialization

### Supported Kafka Versions

This implementation supports Kafka brokers with SSL/TLS enabled. Tested with:
- Confluent Kafka 7.x
- Apache Kafka 2.8+

### Confluent Kafka Library Configuration

The implementation uses the following Confluent Kafka client configuration properties:

- `security.protocol`: Set to `SSL` or `SASL_SSL`
- `ssl.ca.location`: Path to CA certificate
- `ssl.certificate.location`: Path to client certificate
- `ssl.key.location`: Path to client private key

## Additional Resources

- [Confluent Kafka SSL Configuration](https://docs.confluent.io/kafka-clients/python/current/overview.html#ssl-tls-support)
- [Apache Kafka Security Documentation](https://kafka.apache.org/documentation/#security)
- [Kubernetes Secrets Documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
