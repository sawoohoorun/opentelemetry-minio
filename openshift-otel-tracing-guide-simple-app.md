# Complete Guide: OpenTelemetry and Distributed Tracing on OpenShift 4.18

This comprehensive guide walks through the deployment of Red Hat build of OpenTelemetry and Distributed Tracing on OpenShift Container Platform 4.18, including a Java Spring Boot application for testing.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Installation Steps](#installation-steps)
4. [Java Spring Boot Test Application](#java-spring-boot-test-application)
5. [Verification and Testing](#verification-and-testing)
6. [Troubleshooting](#troubleshooting)

## Architecture Overview

The distributed tracing stack consists of the following components:

- **Red Hat build of OpenTelemetry Operator**: Manages OpenTelemetry Collectors
- **OpenTelemetry Collector**: Receives, processes, and exports telemetry data
- **Tempo Operator**: Manages Tempo instances for trace storage
- **TempoStack**: Stores and queries trace data
- **Jaeger UI**: Provides visualization for traces

## Prerequisites

- OpenShift Container Platform 4.18 cluster
- Cluster admin access
- `oc` CLI tool installed and configured
- Helm 3.x installed (for MinIO deployment)
- Podman installed (instead of Docker)

**Important**: While this guide uses Jaeger UI for trace visualization, note that Jaeger UI is deprecated. For production use, consider using the OpenShift Console distributed tracing UI plugin instead (covered in Step 4 of Verification and Testing).

## Installation Steps

### Step 1: Create Project Namespaces

```bash
# Create namespace for operators
oc create namespace openshift-distributed-tracing

# Create namespace for tracing components
oc create namespace tracing-system

# Create namespace for the test application
oc create namespace tracing-demo
```

### Step 2: Install Red Hat build of OpenTelemetry Operator

#### Option A: Using Web Console

1. Navigate to **Operators → OperatorHub**
2. Search for "Red Hat build of OpenTelemetry"
3. Click on the operator and select **Install**
4. Choose:
   - Installation mode: **All namespaces on the cluster**
   - Installed Namespace: **openshift-distributed-tracing**
   - Update Channel: **stable**
   - Approval Strategy: **Automatic**
5. Click **Install**

#### Option B: Using CLI

```yaml
# opentelemetry-operator-subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: opentelemetry-product
  namespace: openshift-distributed-tracing
spec:
  channel: stable
  installPlanApproval: Automatic
  name: opentelemetry-product
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

```bash
oc apply -f opentelemetry-operator-subscription.yaml
```

### Step 3: Install Tempo Operator

#### Option A: Using Web Console

1. Navigate to **Operators → OperatorHub**
2. Search for "Tempo Operator"
3. Click on the operator and select **Install**
4. Choose:
   - Installation mode: **All namespaces on the cluster**
   - Update Channel: **stable**
   - Approval Strategy: **Automatic**
5. Click **Install**

#### Option B: Using CLI

```yaml
# tempo-operator-subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: tempo-product
  namespace: openshift-tempo-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: tempo-product
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

```bash
oc apply -f tempo-operator-subscription.yaml
```

### Step 4: Configure Object Storage

For this example, we'll deploy MinIO using the official Helm chart with self-signed certificates. For production, use S3 or Red Hat OpenShift Data Foundation.

#### Create Security Context Constraints for MinIO

```yaml
# minio-scc.yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: minio-scc
  annotations:
    kubernetes.io/description: "SCC for MinIO Operator"
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities: null
defaultAddCapabilities: null
fsGroup:
  type: RunAsAny
groups: []
priority: 10
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
# Important: Allow seccomp profiles
seccompProfiles:
- '*'
users: []
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
```

```bash
# Apply the SCC
oc apply -f minio-scc.yaml
```

#### Deploy MinIO Operator

```bash
# Create namespace for MinIO Operator
oc new-project minio-operator

# Apply the SCC
oc apply -f minio-scc.yaml

# Add MinIO Operator Helm repository
helm repo add minio-operator https://operator.min.io
helm repo update

# Install MinIO Operator
helm install \
  --namespace minio-operator \
  operator minio-operator/operator

# Wait for operator to be ready
oc wait --for=condition=Ready pods -l app.kubernetes.io/name=operator -n minio-operator --timeout=300s

# Add SCC to MinIO Operator service account (created by Helm)
oc adm policy add-scc-to-user minio-scc -z minio-operator -n minio-operator
```

#### Deploy MinIO Tenant

```bash
# Create namespace for MinIO Tenant
oc new-project minio-tenant

# Create service account for MinIO tenant
oc create serviceaccount minio-sa -n minio-tenant

# Add SCC to MinIO tenant service account
oc adm policy add-scc-to-user minio-scc -z minio-sa -n minio-tenant

# Create credentials secret for MinIO (must have specific keys)
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: minio-creds-secret
  namespace: minio-tenant
type: Opaque
stringData:
  config.env: |
    export MINIO_ROOT_USER=minioadmin
    export MINIO_ROOT_PASSWORD=minioadmin123
EOF

# Create console secret for MinIO
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: minio-console-secret
  namespace: minio-tenant
type: Opaque
stringData:
  CONSOLE_ACCESS_KEY: consoleadmin
  CONSOLE_SECRET_KEY: consoleadmin123
EOF

# Deploy MinIO Tenant
cat <<EOF | oc apply -f -
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: minio
  namespace: minio-tenant
spec:
  pools:
  - name: pool-0
    servers: 4
    volumesPerServer: 1
    volumeClaimTemplate:
      metadata:
        name: data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
      runAsNonRoot: true
      fsGroup: 1000
  
  mountPath: /export
  
  serviceAccountName: minio-sa
  
  # Use auto-generated certificates
  requestAutoCert: true
  
  features:
    bucketDNS: false
    enableSFTP: false
  
  configuration:
    name: minio-creds-secret
  
  users:
  - name: minio-console-secret
  
  env:
  - name: MINIO_BROWSER
    value: "on"
  - name: MINIO_PROMETHEUS_AUTH_TYPE
    value: "public"
  - name: MINIO_STORAGE_CLASS_STANDARD
    value: "EC:2"
  
  # Disable metrics for now
  metrics:
    enabled: false
    
  # Resource allocation
  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 1Gi
      cpu: 500m
EOF

# Wait for MinIO to be ready
oc wait --for=condition=Ready pod -l v1.min.io/tenant=minio -n minio-tenant --timeout=300s

# Extract the auto-generated certificate for Tempo
# For self-signed certificates, the public.crt acts as both the certificate and CA
oc get secret minio-tls -n minio-tenant -o jsonpath='{.data.public\.crt}' | base64 -d > ca.crt

# Create CA ConfigMap for Tempo in tracing-system namespace
# Note: TempoStack expects the key to be named 'service-ca.crt'
oc create configmap minio-ca-bundle \
  --from-file=service-ca.crt=ca.crt \
  -n tracing-system
```

#### Expose MinIO Service and Create Initial Bucket

The MinIO Tenant automatically creates the necessary services. Let's verify they exist and create routes for external access:

```bash
# Verify MinIO services are created
oc get svc -n minio-tenant

# Create a Route to expose MinIO API (optional, for external S3 access)
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: minio
  namespace: minio-tenant
spec:
  to:
    kind: Service
    name: minio
  port:
    targetPort: 443
  tls:
    termination: passthrough
EOF

# Create a Route to expose MinIO Console
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: minio-console
  namespace: minio-tenant
spec:
  to:
    kind: Service
    name: minio-console
  port:
    targetPort: 9443
  tls:
    termination: passthrough
EOF

# Get the console URL
echo "MinIO Console URL: https://$(oc get route minio-console -n minio-tenant -o jsonpath='{.spec.host}')"

# Get console credentials
echo "Console Username: $(oc get secret minio-console-secret -n minio-tenant -o jsonpath='{.data.CONSOLE_ACCESS_KEY}' | base64 -d)"
echo "Console Password: $(oc get secret minio-console-secret -n minio-tenant -o jsonpath='{.data.CONSOLE_SECRET_KEY}' | base64 -d)"

# Create bucket using MinIO client from inside a pod
cat <<EOF | oc apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: create-bucket
  namespace: minio-tenant
spec:
  template:
    spec:
      serviceAccountName: minio-sa
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        runAsNonRoot: true
      containers:
      - name: mc
        image: minio/mc:latest
        command:
        - /bin/sh
        - -c
        - |
          export MC_CONFIG_DIR=/tmp/.mc
          export MC_HOST_minio=https://minioadmin:minioadmin123@minio:443
          
          # Wait for MinIO to be ready
          echo "Waiting for MinIO to be ready..."
          for i in {1..30}; do
            if mc ls minio --insecure 2>/dev/null; then
              echo "MinIO is ready!"
              break
            fi
            echo "Attempt $i/30: MinIO not ready yet, waiting..."
            sleep 10
          done
          
          # Create the bucket
          echo "Creating bucket tempo-traces..."
          mc mb minio/tempo-traces --insecure || true
          
          # List buckets to verify
          echo "Listing buckets..."
          mc ls minio --insecure
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        env:
        - name: HOME
          value: "/tmp"
      volumes:
      - name: tmp-volume
        emptyDir: {}
      restartPolicy: Never
  backoffLimit: 3
EOF

# Wait for job to complete
oc wait --for=condition=complete job/create-bucket -n minio-tenant --timeout=300s

# Check the job logs to verify bucket creation
oc logs -n minio-tenant job/create-bucket
```

### Step 5: Create Storage Secret

```yaml
# tempo-storage-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: tempo-storage-secret
  namespace: tracing-system
type: Opaque
stringData:
  endpoint: https://minio.minio-tenant.svc.cluster.local:443
  bucket: tempo-traces
  access_key_id: minioadmin
  access_key_secret: minioadmin123
```

```bash
# Create the secret
oc apply -f tempo-storage-secret.yaml
```

### Step 6: Deploy TempoStack

Deploy TempoStack with multi-tenancy support:

```yaml
# tempostack.yaml
apiVersion: tempo.grafana.com/v1alpha1
kind: TempoStack
metadata:
  name: tracing-tempo
  namespace: tracing-system
spec:
  storage:
    secret:
      name: tempo-storage-secret
      type: s3
    tls:
      enabled: true
      caName: minio-ca-bundle
  storageSize: 10Gi
  template:
    queryFrontend:
      jaegerQuery:
        enabled: true
    gateway:
      enabled: true
  tenants:
    mode: openshift
    authentication:
      - tenantName: tracing-demo
        tenantId: tracing-demo
      - tenantName: dev
        tenantId: dev
      - tenantName: prod
        tenantId: prod
```

```bash
oc apply -f tempostack.yaml
```

**Note**: Using OpenShift mode for multi-tenancy, which integrates with OpenShift's built-in authentication and authorization.

### Step 7: Deploy OpenTelemetry Collector

For multi-tenant setup with OpenShift authentication, deploy the collector with proper RBAC and configuration:

#### 7.1 Create ServiceAccount and Permissions

```bash
# Create ServiceAccount for the collector
oc create serviceaccount otel-collector-sa -n tracing-demo

# Create ClusterRole for trace writing
cat <<EOF | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tempostack-traces-writer
rules:
- apiGroups:
  - 'tempo.grafana.com'
  resources:
  - tracing-demo  # Must match tenant name in TempoStack
  resourceNames:
  - traces
  verbs:
  - 'create'
EOF

# Bind the role to ServiceAccount
oc adm policy add-cluster-role-to-user tempostack-traces-writer -z otel-collector-sa -n tracing-demo
```

#### 7.2 Deploy the Collector

```yaml
# otel-collector-tracing-demo.yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: tracing-demo
spec:
  mode: deployment
  serviceAccount: otel-collector-sa
  config:
    extensions:
      bearertokenauth:
        filename: "/var/run/secrets/kubernetes.io/serviceaccount/token"
    
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      jaeger:
        protocols:
          grpc:
            endpoint: 0.0.0.0:14250
          thrift_binary:
            endpoint: 0.0.0.0:6832
          thrift_compact:
            endpoint: 0.0.0.0:6831
          thrift_http:
            endpoint: 0.0.0.0:14268
      zipkin:
        endpoint: 0.0.0.0:9411
    
    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 25
    
    exporters:
      otlp:
        endpoint: tempo-tracing-tempo-gateway.tracing-system.svc.cluster.local:8090
        tls:
          insecure: false
          ca_file: "/var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt"
        auth:
          authenticator: bearertokenauth
        headers:
          X-Scope-OrgID: "tracing-demo"
      debug:
        verbosity: detailed
        sampling_initial: 5
        sampling_thereafter: 200
    
    service:
      extensions: [bearertokenauth]
      pipelines:
        traces:
          receivers: [otlp, jaeger, zipkin]
          processors: [memory_limiter, batch]
          exporters: [otlp, debug]
```

```bash
# Deploy collector in tracing-demo namespace
oc apply -f otel-collector-tracing-demo.yaml

# Verify the collector is running
oc get pods -n tracing-demo -l app.kubernetes.io/component=opentelemetry-collector

# Check collector logs
oc logs -f -l app.kubernetes.io/component=opentelemetry-collector -n tracing-demo
```

**Important Notes**:
- The collector uses the **gateway endpoint** on port 8090 for multi-tenant access
- **Bearer token authentication** is required via the ServiceAccount token
- The **X-Scope-OrgID** header must match the tenant name in TempoStack
- **TLS is mandatory** but the service CA certificate is automatically available in pods
- The collector can receive traces via OTLP, Jaeger, or Zipkin protocols

## Java Spring Boot Test Application

### Step 1: Create Spring Boot Application from Spring Initializr

1. Go to [https://start.spring.io/](https://start.spring.io/)

2. Configure your project:
   - **Project**: Maven
   - **Language**: Java
   - **Spring Boot**: 3.2.0 (or latest 3.x)
   - **Group**: com.example
   - **Artifact**: tracing-demo
   - **Name**: tracing-demo
   - **Package name**: com.example.tracingdemo
   - **Packaging**: Jar
   - **Java**: 17

3. Add Dependencies (click "ADD DEPENDENCIES"):
   - Spring Web
   - Spring Boot Actuator

4. Click "GENERATE" to download the project

5. Extract the downloaded ZIP file

**Note**: Spring Initializr uses dots in package names by default. If you prefer underscores (e.g., `com.example.tracing_demo`), you'll need to rename the packages after generation.

### Step 2: Add OpenTelemetry Dependencies

Update your `pom.xml` to add OpenTelemetry dependencies:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>tracing-demo</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    
    <properties>
        <java.version>17</java.version>
        <opentelemetry.version>1.32.0</opentelemetry.version>
        <opentelemetry-instrumentation.version>1.32.0-alpha</opentelemetry-instrumentation.version>
        <opentelemetry-semconv.version>1.21.0-alpha</opentelemetry-semconv.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
        <!-- OpenTelemetry dependencies -->
        <dependency>
            <groupId>io.opentelemetry</groupId>
            <artifactId>opentelemetry-api</artifactId>
            <version>${opentelemetry.version}</version>
        </dependency>
        
        <dependency>
            <groupId>io.opentelemetry</groupId>
            <artifactId>opentelemetry-sdk</artifactId>
            <version>${opentelemetry.version}</version>
        </dependency>
        
        <dependency>
            <groupId>io.opentelemetry</groupId>
            <artifactId>opentelemetry-exporter-otlp</artifactId>
            <version>${opentelemetry.version}</version>
        </dependency>
        
        <dependency>
            <groupId>io.opentelemetry</groupId>
            <artifactId>opentelemetry-semconv</artifactId>
            <version>${opentelemetry-semconv.version}</version>
        </dependency>
        
        <!-- OpenTelemetry Spring Boot Instrumentation -->
        <dependency>
            <groupId>io.opentelemetry.instrumentation</groupId>
            <artifactId>opentelemetry-spring-boot-starter</artifactId>
            <version>${opentelemetry-instrumentation.version}</version>
        </dependency>
        
        <!-- Test Dependencies -->
        <!-- JUnit 5 -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- Spring Boot Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### Step 3: Application Configuration

Replace the contents of `src/main/resources/application.yaml` (or create it if you have application.properties):

```yaml
# src/main/resources/application.yaml
spring:
  application:
    name: tracing-demo

server:
  port: 8080

# OpenTelemetry Configuration for multi-tenant setup
otel:
  exporter:
    otlp:
      endpoint: http://otel-collector-collector:4317
      protocol: grpc
  resource:
    attributes:
      service.name: ${spring.application.name}
      service.namespace: tracing-demo
      deployment.environment: production
      tenant: tracing-demo
  instrumentation:
    spring-boot:
      enabled: true
  metrics:
    export:
      enabled: false
  logs:
    export:
      enabled: false

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  tracing:
    sampling:
      probability: 1.0
```

### Step 4: Main Application Class

The main application class is already created by Spring Initializr. Update the package name if needed:

```java
// src/main/java/com/example/tracing_demo/TracingDemoApplication.java
package com.example.tracing_demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class TracingDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(TracingDemoApplication.class, args);
    }
}
```

### Step 5: Controller with Tracing

Create a new controller class:

```java
// src/main/java/com/example/tracing_demo/controller/DemoController.java
package com.example.tracing_demo.controller;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;

import java.util.Random;
import java.util.concurrent.TimeUnit;

@RestController
@RequestMapping("/api")
public class DemoController {
    
    private static final Logger logger = LoggerFactory.getLogger(DemoController.class);
    private final Tracer tracer;
    private final RestTemplate restTemplate;
    private final Random random = new Random();
    
    public DemoController() {
        this.tracer = GlobalOpenTelemetry.getTracer(
            "com.example.tracing_demo", "1.0.0");
        this.restTemplate = new RestTemplate();
    }
    
    @GetMapping("/hello")
    public ResponseEntity<String> hello(@RequestParam(defaultValue = "World") String name) {
        Span span = tracer.spanBuilder("hello-endpoint")
            .setSpanKind(SpanKind.SERVER)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            span.setAttribute("user.name", name);
            
            // Simulate some processing
            processRequest(name);
            
            String response = String.format("Hello, %s! Tracing is working!", name);
            span.setAttribute("response.message", response);
            
            return ResponseEntity.ok(response);
        } finally {
            span.end();
        }
    }
    
    @GetMapping("/chain")
    public ResponseEntity<String> chain() {
        Span span = tracer.spanBuilder("chain-endpoint")
            .setSpanKind(SpanKind.SERVER)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // First service call
            String result1 = callService("service-1");
            
            // Second service call
            String result2 = callService("service-2");
            
            // Third service call
            String result3 = callService("service-3");
            
            String response = String.format("Chain complete: %s -> %s -> %s", 
                result1, result2, result3);
            
            return ResponseEntity.ok(response);
        } finally {
            span.end();
        }
    }
    
    @GetMapping("/error")
    public ResponseEntity<String> error() {
        Span span = tracer.spanBuilder("error-endpoint")
            .setSpanKind(SpanKind.SERVER)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // Simulate random errors
            if (random.nextBoolean()) {
                span.recordException(new RuntimeException("Simulated error"));
                span.setStatus(io.opentelemetry.api.trace.StatusCode.ERROR);
                throw new RuntimeException("Simulated error for testing");
            }
            
            return ResponseEntity.ok("No error occurred this time!");
        } finally {
            span.end();
        }
    }
    
    private void processRequest(String name) {
        Span span = tracer.spanBuilder("process-request")
            .setSpanKind(SpanKind.INTERNAL)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            span.setAttribute("processing.name", name);
            
            // Simulate processing time
            int processingTime = random.nextInt(100) + 50;
            span.setAttribute("processing.time.ms", processingTime);
            
            try {
                TimeUnit.MILLISECONDS.sleep(processingTime);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                span.recordException(e);
            }
            
            // Simulate database call
            databaseQuery(name);
            
        } finally {
            span.end();
        }
    }
    
    private void databaseQuery(String parameter) {
        Span span = tracer.spanBuilder("database-query")
            .setSpanKind(SpanKind.CLIENT)
            .setAttribute("db.system", "postgresql")
            .setAttribute("db.operation", "SELECT")
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            span.setAttribute("db.statement", 
                String.format("SELECT * FROM users WHERE name = '%s'", parameter));
            
            // Simulate database latency
            try {
                TimeUnit.MILLISECONDS.sleep(random.nextInt(50) + 10);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                span.recordException(e);
            }
        } finally {
            span.end();
        }
    }
    
    private String callService(String serviceName) {
        Span span = tracer.spanBuilder("call-" + serviceName)
            .setSpanKind(SpanKind.CLIENT)
            .setAttribute("peer.service", serviceName)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // Simulate service call latency
            int latency = random.nextInt(200) + 50;
            span.setAttribute("http.latency.ms", latency);
            
            try {
                TimeUnit.MILLISECONDS.sleep(latency);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                span.recordException(e);
            }
            
            return serviceName + "-response";
        } finally {
            span.end();
        }
    }
}
```

### Step 6: Health Check Endpoint

Create a health controller:

```java
// src/main/java/com/example/tracing_demo/controller/HealthController.java
package com.example.tracing_demo.controller;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@RestController
public class HealthController {
    
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        Map<String, String> status = new HashMap<>();
        status.put("status", "UP");
        status.put("service", "tracing-demo");
        return ResponseEntity.ok(status);
    }
}
```

### Step 7: Containerfile (Dockerfile with Podman)

Create a Containerfile in the project root:

```dockerfile
# Containerfile
FROM registry.access.redhat.com/ubi9/openjdk-17-runtime:latest

# Red Hat UBI images run as user 185 by default
USER root

# Create application directory
WORKDIR /deployments

# Copy the application JAR
COPY target/tracing-demo-*.jar app.jar

# Set permissions for the default user (185)
RUN chown -R 185:0 /deployments && \
    chmod -R g=u /deployments

# Switch back to the default user
USER 185

EXPOSE 8080

# The UBI image automatically runs the Java application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Alternative: Using Red Hat UBI Minimal with OpenJDK

If you prefer a smaller image size, you can use the UBI minimal variant:

```dockerfile
# Containerfile-minimal
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest

# Install Java 17 runtime
RUN microdnf install -y java-17-openjdk-headless && \
    microdnf clean all

# Create non-root user
RUN useradd -u 1001 -g 0 -s /usr/sbin/nologin appuser

WORKDIR /app

COPY target/tracing-demo-*.jar app.jar

RUN chown -R 1001:0 /app && \
    chmod -R g=u /app

USER 1001

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Step 7.1: Create Unit Tests (Optional)

Since we've added test dependencies, here's an example test class:

```java
// src/test/java/com/example/tracing_demo/controller/DemoControllerTest.java
package com.example.tracing_demo.controller;

import com.example.tracing_demo.TracingDemoApplication;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;

@SpringBootTest(classes = TracingDemoApplication.class)
@AutoConfigureMockMvc
class DemoControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void testHelloEndpoint() throws Exception {
        mockMvc.perform(get("/api/hello?name=Test"))
                .andExpect(status().isOk())
                .andExpect(content().string("Hello, Test! Tracing is working!"));
    }

    @Test
    void testHealthEndpoint() throws Exception {
        mockMvc.perform(get("/health"))
                .andExpect(status().isOk())
                .andExpect(content().json("{\"status\":\"UP\",\"service\":\"tracing-demo\"}"));
    }

    @Test
    void testChainEndpoint() throws Exception {
        mockMvc.perform(get("/api/chain"))
                .andExpect(status().isOk())
                .andExpect(content().string(
                    "Chain complete: service-1-response -> service-2-response -> service-3-response"));
    }
}
```

### Step 8: Build and Deploy the Application with Podman

```bash
# Build the application
./mvnw clean package

# Run tests (optional)
./mvnw test

# Build container image with Podman
podman build -t tracing-demo:1.0.0 -f Containerfile .

# Tag for your registry
podman tag tracing-demo:1.0.0 your-registry/tracing-demo:1.0.0

# Login to your registry (if needed)
podman login your-registry

# Push to registry
podman push your-registry/tracing-demo:1.0.0

# Optional: Save image to a tar file for transfer
podman save -o tracing-demo.tar tracing-demo:1.0.0

# Optional: Load image on another system
podman load -i tracing-demo.tar
```

#### Alternative: Build directly in OpenShift using Source-to-Image (S2I)

If you prefer to build directly in OpenShift without using Podman:

```bash
# Create a binary build
oc new-build --name=tracing-demo --binary=true --docker-image=registry.access.redhat.com/ubi8/openjdk-17:latest -n tracing-demo

# Start the build with your JAR file
oc start-build tracing-demo --from-file=target/tracing-demo-1.0.0.jar --follow -n tracing-demo

# Create the application from the built image
oc new-app tracing-demo -n tracing-demo

# Expose the service
oc expose svc/tracing-demo -n tracing-demo
```

### Step 9: Deploy to OpenShift

```yaml
# tracing-demo-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tracing-demo
  namespace: tracing-demo
  labels:
    app: tracing-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tracing-demo
  template:
    metadata:
      labels:
        app: tracing-demo
    spec:
      containers:
      - name: tracing-demo
        image: your-registry/tracing-demo:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: OTEL_SERVICE_NAME
          value: "tracing-demo"
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://otel-collector-collector:4317"
        - name: OTEL_EXPORTER_OTLP_PROTOCOL
          value: "grpc"
        - name: OTEL_TRACES_EXPORTER
          value: "otlp"
        - name: OTEL_METRICS_EXPORTER
          value: "none"
        - name: OTEL_LOGS_EXPORTER
          value: "none"
        - name: OTEL_RESOURCE_ATTRIBUTES
          value: "tenant=tracing-demo,service.namespace=tracing-demo"
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 250m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: tracing-demo
  namespace: tracing-demo
spec:
  selector:
    app: tracing-demo
  ports:
  - port: 8080
    targetPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: tracing-demo
  namespace: tracing-demo
spec:
  to:
    kind: Service
    name: tracing-demo
  port:
    targetPort: 8080
  tls:
    termination: edge
```

```bash
oc apply -f tracing-demo-deployment.yaml
```

### Step 10: Deploy Additional Tenants (Optional)

To demonstrate multi-tenancy, create another application in a different namespace:

```bash
# Create dev namespace
oc new-project dev

# Create ServiceAccount
oc create serviceaccount otel-collector-sa -n dev

# Grant permissions for dev tenant
cat <<EOF | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tempostack-traces-writer-dev
rules:
- apiGroups:
  - 'tempo.grafana.com'
  resources:
  - dev  # tenant name
  resourceNames:
  - traces
  verbs:
  - 'create'
EOF

# Bind the role
oc adm policy add-cluster-role-to-user tempostack-traces-writer-dev -z otel-collector-sa -n dev

# Create collector for dev tenant
cat <<EOF | oc apply -f -
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: dev
spec:
  mode: deployment
  serviceAccount: otel-collector-sa
  config:
    extensions:
      bearertokenauth:
        filename: "/var/run/secrets/kubernetes.io/serviceaccount/token"
    
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    
    processors:
      batch: {}
    
    exporters:
      otlp:
        endpoint: tempo-tracing-tempo-gateway.tracing-system.svc.cluster.local:8090
        tls:
          insecure: false
          ca_file: "/var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt"
        auth:
          authenticator: bearertokenauth
        headers:
          X-Scope-OrgID: "dev"
    
    service:
      extensions: [bearertokenauth]
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlp]
EOF

# Deploy a test app in dev namespace
# (Use the same Spring Boot app but with environment configured for dev tenant)
```

## Verification and Testing

### Step 1: Verify All Components are Running

```bash
# Check operators
oc get pods -n openshift-distributed-tracing
oc get pods -n openshift-tempo-operator

# Check tracing components
oc get pods -n tracing-system

# Check application
oc get pods -n tracing-demo
```

### Step 2: Get Routes

```bash
# Get Jaeger UI route
oc get route -n tracing-system | grep tempo

# Get application route
oc get route -n tracing-demo
```

### Step 3: Generate Test Traffic

```bash
# Get the application route
APP_ROUTE=$(oc get route tracing-demo -n tracing-demo -o jsonpath='{.spec.host}')

# Test endpoints
curl https://$APP_ROUTE/api/hello?name=OpenShift
curl https://$APP_ROUTE/api/chain
curl https://$APP_ROUTE/api/error

# Generate load
for i in {1..100}; do
  curl https://$APP_ROUTE/api/hello?name=User$i
  curl https://$APP_ROUTE/api/chain
  curl https://$APP_ROUTE/api/error || true
  sleep 1
done
```

### Step 4: Access Jaeger UI

**Note**: Jaeger UI is deprecated and will be removed in a future release. For a better experience, use the OpenShift Console distributed tracing UI plugin (see Alternative Method below).

#### Traditional Jaeger UI Access

For multi-tenant setup with OpenShift mode:

1. Create a route to access Jaeger UI through the gateway:
```bash
# Create route to Tempo Gateway
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: tempo-gateway
  namespace: tracing-system
spec:
  to:
    kind: Service
    name: tempo-tracing-tempo-gateway
  port:
    targetPort: public
  tls:
    termination: edge
EOF

# Get the gateway URL
GATEWAY_URL=$(oc get route tempo-gateway -n tracing-system -o jsonpath='https://{.spec.host}')
echo "Tempo Gateway URL: $GATEWAY_URL"
```

2. Access Jaeger UI for specific tenants:
   - For tracing-demo tenant: `$GATEWAY_URL/api/traces/v1/tracing-demo`
   - For dev tenant: `$GATEWAY_URL/api/traces/v1/dev`
   - For prod tenant: `$GATEWAY_URL/api/traces/v1/prod`
   
   You'll need to authenticate with your OpenShift credentials.

3. Alternative: Direct access to Query Frontend (bypasses multi-tenancy):
```bash
# Create route directly to Query Frontend
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: jaeger-ui-direct
  namespace: tracing-system
spec:
  to:
    kind: Service
    name: tempo-tracing-tempo-query-frontend
  port:
    targetPort: jaeger-ui
  tls:
    termination: edge
EOF

# Get the Jaeger UI URL
echo "Direct Jaeger UI: https://$(oc get route jaeger-ui-direct -n tracing-system -o jsonpath='{.spec.host}')"
```

#### Alternative Method: OpenShift Console Distributed Tracing UI (Recommended)

The modern way to view traces is through the OpenShift Console with the distributed tracing UI plugin:

1. **Install the Cluster Observability Operator** (if not already installed):
```bash
# Create namespace
oc create namespace openshift-observability-operator

# Install Cluster Observability Operator
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-observability-operator
  namespace: openshift-observability-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: cluster-observability-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

2. **Install the Distributed Tracing UI Plugin**:
```bash
# The plugin is typically installed automatically with the Red Hat build of OpenTelemetry
# Verify it's enabled in the console
oc get consoles.operator.openshift.io cluster -o yaml | grep -A 5 plugins

# If not listed, you may need to enable it
oc patch consoles.operator.openshift.io cluster --type='json' -p='[{"op": "add", "path": "/spec/plugins/-", "value": "distributed-tracing"}]'
```

3. **Access Traces in OpenShift Console**:
   - Log into the OpenShift Console
   - Navigate to **Observe → Traces** in the left menu
   - Select your namespace (e.g., `tracing-demo`)
   - View and analyze traces directly in the console

4. **Configure Tempo as the trace backend**:
```bash
# Create a ConfigMap to point to your Tempo instance
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: distributed-tracing-config
  namespace: openshift-config
data:
  config.yaml: |
    query_frontend_endpoint: "http://tempo-tracing-tempo-query-frontend.tracing-system.svc.cluster.local:16686"
EOF
```

**Benefits of using the OpenShift Console**:
- Integrated with OpenShift authentication
- Unified observability experience
- Modern UI with better filtering and search capabilities
- Direct integration with other observability features
- No separate routes or port-forwarding needed

## Troubleshooting

### Common Issues and Solutions

#### 1. Pods Not Starting

```bash
# Check pod events
oc describe pod <pod-name> -n <namespace>

# Check logs
oc logs <pod-name> -n <namespace>
```

#### 2. No Traces Appearing

```bash
# Check collector logs
oc logs -l app.kubernetes.io/component=opentelemetry-collector -n tracing-demo

# Verify the collector is receiving traces (look for debug output)
oc logs -l app.kubernetes.io/component=opentelemetry-collector -n tracing-demo | grep "ResourceSpans"

# Check if Tempo pods are running
oc get pods -n tracing-system

# Check Tempo gateway logs
oc logs -l app.kubernetes.io/component=gateway -n tracing-system

# Verify service connectivity
oc exec -it <app-pod> -n tracing-demo -- curl -v http://otel-collector-collector:4318/v1/traces

# Check authentication
oc exec -it <collector-pod> -n tracing-demo -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Verify RBAC permissions
oc auth can-i create traces --as=system:serviceaccount:tracing-demo:otel-collector-sa --subresource=tempo.grafana.com/tracing-demo
```

#### 3. Authentication/Permission Errors

```bash
# Check if ServiceAccount exists
oc get sa otel-collector-sa -n tracing-demo

# Verify ClusterRole binding
oc get clusterrolebinding -o wide | grep otel-collector-sa

# Check collector logs for 403 errors
oc logs -l app.kubernetes.io/component=opentelemetry-collector -n tracing-demo | grep -E "403|permission|denied"

# Re-apply permissions if needed
oc adm policy add-cluster-role-to-user tempostack-traces-writer -z otel-collector-sa -n tracing-demo
```

#### 3. Storage Issues

```bash
# Check MinIO tenant status
oc get tenant -n minio-tenant
oc describe tenant minio -n minio-tenant

# Check MinIO pods and services
oc get pods -n minio-tenant
oc get svc -n minio-tenant

# Verify service accounts and SCC
oc get sa -n minio-operator
oc get sa -n minio-tenant
oc describe scc minio-scc

# Check SCC assignments
oc adm policy who-can use scc minio-scc -n minio-operator
oc adm policy who-can use scc minio-scc -n minio-tenant

# Check certificates
oc get secret minio-tls -n minio-tenant
oc describe secret minio-tls -n minio-tenant

# Verify CA ConfigMap
oc get configmap minio-ca-bundle -n tracing-system

# Verify storage secret
oc get secret tempo-storage-secret -n tracing-system -o yaml

# Test MinIO connectivity from a pod
oc run -it --rm debug-pod --image=curlimages/curl -n tracing-system -- sh
# Inside the pod:
curl -k https://minio.minio-tenant.svc.cluster.local:443

# Check MinIO route if created
oc get route -n minio-tenant
```

#### 4. Route Issues

```bash
# Check routes
oc get routes -A | grep -E "tempo|tracing"

# Test route connectivity
curl -k https://<route-url>/api/health
```

### Debug Commands

```bash
# Get all resources in tracing-system
oc get all -n tracing-system

# Check TempoStack status
oc describe tempostack tracing-tempo -n tracing-system

# Check OpenTelemetry Collector status
oc describe opentelemetrycollector otel-collector -n tracing-demo

# View recent events
oc get events -n tracing-system --sort-by='.lastTimestamp'
```

## Best Practices

1. **Resource Limits**: Always set appropriate resource limits for all components
2. **Storage**: Use persistent storage for production deployments
3. **Security**: Enable TLS for all communications in production
4. **Sampling**: Adjust sampling rates based on traffic volume
5. **Retention**: Configure appropriate retention policies for trace data
6. **Monitoring**: Set up alerts for component health and performance
7. **Certificate Management**: Rotate self-signed certificates before expiration
8. **Operator Management**: Keep operators updated through their stable channels
9. **Container Images**: Use Podman for building and managing container images locally
10. **Registry Security**: Always use secure registries and scan images for vulnerabilities

## Podman-Specific Tips

1. **Rootless Containers**: Podman runs containers in rootless mode by default, which is more secure
2. **Pod Support**: Use `podman pod` commands to test multi-container applications locally
3. **Compatibility**: Podman commands are largely compatible with Docker commands
4. **No Daemon**: Podman doesn't require a daemon, making it more suitable for CI/CD pipelines
5. **Integration**: Podman integrates well with systemd for container management

## Conclusion

This guide provides a complete implementation of distributed tracing on OpenShift 4.18 using:
- Red Hat build of OpenTelemetry for trace collection
- Tempo for trace storage with multi-tenant support
- Jaeger UI for visualization
- A fully instrumented Java Spring Boot application
- Podman for container image management

The setup includes:
- Multi-tenant configuration with OpenShift authentication
- Per-namespace OpenTelemetry collectors with tenant isolation
- Spring Boot application created from Spring Initializr
- Complete observability stack
- Container build process using Podman instead of Docker

For production deployments, ensure you:
- Use enterprise-grade object storage
- Enable authentication and authorization
- Configure TLS for all communications
- Set up monitoring and alerting
- Implement proper backup and recovery procedures
- Use secure container registries
- Regularly update container images