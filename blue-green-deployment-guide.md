# Blue-Green Deployment for Message-Driven Spring Boot Applications: A Complete Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Understanding Blue-Green Deployment for Message-Driven Applications](#understanding-blue-green-deployment-for-message-driven-applications)
3. [Architecture Overview](#architecture-overview)
4. [Prerequisites](#prerequisites)
5. [Consumer-Based Implementation Strategy](#consumer-based-implementation-strategy)
6. [Step-by-Step Blue-Green Deployment Process](#step-by-step-blue-green-deployment-process)
7. [Handling Database Migrations](#handling-database-migrations)
8. [Message Queue Consumer Management](#message-queue-consumer-management)
9. [Alternative Deployment Strategies](#alternative-deployment-strategies)
10. [Best Practices](#best-practices)
11. [Monitoring and Rollback](#monitoring-and-rollback)
12. [Conclusion](#conclusion)

## Introduction

Deploying message-driven applications presents unique challenges compared to traditional HTTP-based services. For Spring Boot applications that exclusively process messages from IBM MQ and Apache Kafka without exposing HTTP endpoints for business logic, the deployment strategy must focus on consumer management rather than traffic routing.

This comprehensive guide explores blue-green deployment specifically designed for message-driven Java Spring Boot applications that integrate with IBM MQ, Apache Kafka, and CockroachDB, all orchestrated within a Kubernetes environment.

## Understanding Blue-Green Deployment for Message-Driven Applications

Blue-green deployment for message-driven applications differs significantly from HTTP-based services. Instead of routing traffic through load balancers, we manage message consumption by controlling which environment's consumers are active.

### Core Principles for Message-Driven Applications

- **Consumer Switching**: Activate/deactivate message consumers instead of routing traffic
- **Message Ordering**: Ensure message processing order is maintained during transitions
- **Zero Message Loss**: Guarantee no messages are lost during deployment
- **Poison Message Handling**: Manage failed messages appropriately during version changes

### Visual Representation

```
┌─────────────────────────────────────────────────────────────────┐
│              Message-Driven Blue-Green Deployment              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌─────────────────┐   │
│  │              │    │              │    │                 │   │
│  │   IBM MQ     │    │ Apache Kafka │    │   Blue Env      │   │
│  │              │────│              │────│  (Active        │   │
│  │ ┌──────────┐ │    │ ┌──────────┐ │    │   Consumers)    │   │
│  │ │ Queue A  │ │    │ │ Topic X  │ │    │                 │   │
│  │ │ Queue B  │ │    │ │ Topic Y  │ │    │ ┌─────────────┐ │   │
│  │ └──────────┘ │    │ └──────────┘ │    │ │  Consumer   │ │   │
│  └──────────────┘    └──────────────┘    │ │   Groups    │ │   │
│                                           │ └─────────────┘ │   │
│                                           └─────────────────┘   │
│                                                                 │
│                       During Deployment:                       │
│                                                                 │
│                                           ┌─────────────────┐   │
│                       Switch Consumers    │                 │   │
│                            to Green  ────▶│   Green Env     │   │
│                                           │  (New Active    │   │
│                                           │   Consumers)    │   │
│                                           │                 │   │
│                                           │ ┌─────────────┐ │   │
│                                           │ │  Consumer   │ │   │
│                                           │ │   Groups    │ │   │
│                                           │ └─────────────┘ │   │
│                                           └─────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Architecture Overview

Our message-driven Spring Boot application architecture focuses on asynchronous message processing rather than HTTP request handling:

```
┌─────────────────────────────────────────────────────────────────┐
│              Message-Driven Application Architecture            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌─────────────────────────────────────┐   │
│  │              │    │                                     │   │
│  │ Message      │    │         Kubernetes Cluster         │   │
│  │ Producers    │    │                                     │   │
│  │ (External)   │    │  ┌─────────────┐  ┌─────────────┐  │   │
│  └──────────────┘    │  │    Blue     │  │   Green     │  │   │
│         │             │  │ Environment │  │Environment  │  │   │
│         │             │  │             │  │             │  │   │
│         ▼             │  │ ┌─────────┐ │  │ ┌─────────┐ │  │   │
│  ┌──────────────┐    │  │ │SpringBoot│ │  │ │SpringBoot│ │  │   │
│  │              │    │  │ │Message   │ │  │ │Message   │ │  │   │
│  │   IBM MQ     │────┼──│ │Consumer  │ │  │ │Consumer  │ │  │   │
│  │              │    │  │ │   App    │ │  │ │   App    │ │  │   │
│  └──────────────┘    │  │ └─────────┘ │  │ └─────────┘ │  │   │
│                       │  │             │  │             │  │   │
│  ┌──────────────┐    │  │ ┌─────────┐ │  │ ┌─────────┐ │  │   │
│  │              │    │  │ │Consumer │ │  │ │Consumer │ │  │   │
│  │Apache Kafka  │────┼──│ │Groups   │ │  │ │Groups   │ │  │   │
│  │              │    │  │ │(Active) │ │  │ │(Standby)│ │  │   │
│  └──────────────┘    │  │ └─────────┘ │  │ └─────────┘ │  │   │
│                       │  └─────────────┘  └─────────────┘  │   │
│                       └─────────────────────────────────────┘   │
│                                       │                         │
│                                       │                         │
│                                       ▼                         │
│                               ┌─────────────────┐               │
│                               │                 │               │
│                               │   CockroachDB   │               │
│                               │   (Shared)      │               │
│                               │                 │               │
│                               └─────────────────┘               │
│                                                                 │
│  Key Difference: No HTTP ingress for business logic            │
│  Deployment switching happens at consumer group level          │
└─────────────────────────────────────────────────────────────────┘
```

### Key Components

1. **Spring Boot Message Consumer Application**: Processes messages from queues/topics
2. **IBM MQ**: Message queuing with queue-based consumption
3. **Apache Kafka**: Event streaming with consumer group management
4. **CockroachDB**: Shared database for both environments
5. **Kubernetes**: Container orchestration platform
6. **Consumer Group Management**: Controls which environment processes messages

## Prerequisites

Before implementing blue-green deployment, ensure you have:

### Infrastructure Requirements
- Kubernetes cluster with sufficient resources for two environments
- Persistent storage for CockroachDB
- Network policies for service isolation
- Monitoring and logging infrastructure

### Application Requirements
- Message-driven architecture (no HTTP business endpoints)
- Consumer group management capabilities
- Idempotent message processing
- Dead letter queue handling
- Graceful consumer shutdown
- Health check endpoints (`/actuator/health`) for monitoring only
- Database transaction management
- Configuration externalization

### Tools and Technologies
```bash
# Required tools
kubectl          # Kubernetes CLI
helm            # Kubernetes package manager
kustomize       # Kubernetes configuration management
kafka-tools     # Kafka consumer group management
prometheus      # Monitoring
grafana         # Visualization
```

## Consumer-Based Implementation Strategy

### 1. Consumer Group Strategy

The key to message-driven blue-green deployment is managing consumer groups and ensuring only one environment processes messages at a time.

```yaml
# Blue Environment Consumer Configuration
spring:
  kafka:
    consumer:
      group-id: payment-processor-blue
      auto-offset-reset: earliest
      enable-auto-commit: false
      
  jms:
    ibm:
      mq:
        queue-manager: QM1
        channel: DEV.APP.SVRCONN
        conn-name: ibm-mq-service(1414)
        user: app
        consumer-group: payment-processor-blue
```

```yaml
# Green Environment Consumer Configuration  
spring:
  kafka:
    consumer:
      group-id: payment-processor-green
      auto-offset-reset: earliest
      enable-auto-commit: false
      
  jms:
    ibm:
      mq:
        queue-manager: QM1
        channel: DEV.APP.SVRCONN
        conn-name: ibm-mq-service(1414)
        user: app
        consumer-group: payment-processor-green
```

### 2. Consumer Lifecycle Management

```java
@Component
public class ConsumerLifecycleManager {
    
    @Autowired
    private KafkaListenerEndpointRegistry kafkaRegistry;
    
    @Autowired
    private JmsListenerEndpointRegistry jmsRegistry;
    
    @Value("${app.consumer.active:false}")
    private boolean consumerActive;
    
    @PostConstruct
    public void initializeConsumers() {
        if (consumerActive) {
            startConsumers();
        } else {
            stopConsumers();
        }
    }
    
    public void startConsumers() {
        kafkaRegistry.getListenerContainers().forEach(container -> {
            if (!container.isRunning()) {
                container.start();
            }
        });
        
        jmsRegistry.getListenerContainers().forEach(container -> {
            if (!container.isRunning()) {
                container.start();
            }
        });
    }
    
    public void stopConsumers() {
        kafkaRegistry.getListenerContainers().forEach(container -> {
            if (container.isRunning()) {
                container.stop();
            }
        });
        
        jmsRegistry.getListenerContainers().forEach(container -> {
            if (container.isRunning()) {
                container.stop();
            }
        });
    }
}
```

### 3. Environment Labeling Strategy

```yaml
# Blue Environment Labels
metadata:
  labels:
    app: message-processor
    version: blue
    environment: production
    slot: blue
    consumer-active: "true"  # Only blue is active initially

# Green Environment Labels
metadata:
  labels:
    app: message-processor
    version: green
    environment: production
    slot: green
    consumer-active: "false"  # Green starts inactive
```

## Step-by-Step Blue-Green Deployment Process

### Phase 1: Preparation

#### 1.1 Current State Assessment

```bash
# Check current active environment
kubectl get services -l app=spring-boot-app
kubectl get deployments -l app=spring-boot-app
kubectl describe service spring-boot-service | grep Selector
```

#### 1.2 Database Backup

```bash
# Create database backup before deployment
kubectl exec -it cockroachdb-0 -- /cockroach/cockroach sql \
  --execute="BACKUP DATABASE myapp TO 'gs://backup-bucket/pre-deployment-$(date +%Y%m%d-%H%M%S)';"
```

### Phase 2: Green Environment Deployment

#### 2.1 Deploy New Version to Green Environment (Consumers Inactive)

```yaml
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: message-processor-green
  labels:
    app: message-processor
    slot: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: message-processor
      slot: green
  template:
    metadata:
      labels:
        app: message-processor
        slot: green
    spec:
      containers:
      - name: message-processor
        image: myregistry/message-processor:v1.1.0
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: APP_CONSUMER_ACTIVE
          value: "false"  # Start with consumers disabled
        - name: KAFKA_CONSUMER_GROUP_ID
          value: "payment-processor-green"
        - name: IBM_MQ_CONSUMER_GROUP
          value: "payment-processor-green"
        - name: DB_HOST
          value: "cockroachdb-service"
        - name: KAFKA_BROKERS
          value: "kafka-service:9092"
        - name: IBM_MQ_HOST
          value: "ibm-mq-service"
        ports:
        - containerPort: 8080  # Management endpoints only
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

#### 2.2 Apply Green Deployment

```bash
# Deploy green environment with inactive consumers
kubectl apply -f green-deployment.yaml

# Wait for deployment to be ready
kubectl rollout status deployment/message-processor-green --timeout=300s

# Verify pods are running (consumers should be inactive)
kubectl get pods -l slot=green

# Verify consumers are not active
kubectl logs -l slot=green | grep "Consumer.*stopped"
```

### Phase 3: Testing and Validation

#### 3.1 Consumer Activation Test Service

Create a management endpoint to control consumer activation:

```java
@RestController
@RequestMapping("/management")
public class ConsumerManagementController {
    
    @Autowired
    private ConsumerLifecycleManager consumerManager;
    
    @PostMapping("/consumers/start")
    public ResponseEntity<String> startConsumers() {
        try {
            consumerManager.startConsumers();
            return ResponseEntity.ok("Consumers started successfully");
        } catch (Exception e) {
            return ResponseEntity.status(500).body("Failed to start consumers: " + e.getMessage());
        }
    }
    
    @PostMapping("/consumers/stop")
    public ResponseEntity<String> stopConsumers() {
        try {
            consumerManager.stopConsumers();
            return ResponseEntity.ok("Consumers stopped successfully");
        } catch (Exception e) {
            return ResponseEntity.status(500).body("Failed to stop consumers: " + e.getMessage());
        }
    }
    
    @GetMapping("/consumers/status")
    public ResponseEntity<Map<String, Object>> getConsumerStatus() {
        Map<String, Object> status = consumerManager.getConsumerStatus();
        return ResponseEntity.ok(status);
    }
}
```

#### 3.2 Test Message Processing in Green Environment

```bash
# Temporarily activate consumers in green environment for testing
GREEN_POD=$(kubectl get pods -l slot=green -o jsonpath='{.items[0].metadata.name}')

# Start consumers for testing
kubectl exec $GREEN_POD -- curl -X POST http://localhost:8080/management/consumers/start

# Send test messages to verify processing
# For Kafka
kubectl exec -it kafka-0 -- kafka-console-producer --topic payment-events --bootstrap-server localhost:9092
# Type test messages...

# For IBM MQ
kubectl exec -it ibm-mq-0 -- /opt/mqm/samp/bin/amqsput TEST.QUEUE QM1
# Type test messages...

# Monitor green environment logs for message processing
kubectl logs -l slot=green -f

# Verify database updates
kubectl exec -it cockroachdb-0 -- /cockroach/cockroach sql \
  --execute="SELECT COUNT(*) FROM processed_messages WHERE processed_by='green';"

# Stop consumers after testing
kubectl exec $GREEN_POD -- curl -X POST http://localhost:8080/management/consumers/stop
```

#### 3.3 Automated Test Suite

```bash
# Run integration tests against green environment
kubectl run test-runner --image=myregistry/integration-tests:latest \
  --env="TARGET_URL=http://spring-boot-service-green-test:8080" \
  --restart=Never

# Monitor test results
kubectl logs test-runner -f
```

### Phase 4: Consumer Switching

#### 4.1 Graceful Consumer Transition

The critical phase involves stopping blue environment consumers and starting green environment consumers with minimal message processing gap.

```bash
#!/bin/bash
# consumer-switch.sh

BLUE_PODS=$(kubectl get pods -l slot=blue -o jsonpath='{.items[*].metadata.name}')
GREEN_PODS=$(kubectl get pods -l slot=green -o jsonpath='{.items[*].metadata.name}')

echo "Starting consumer switch from blue to green..."

# Step 1: Stop accepting new messages in blue environment
echo "Stopping blue consumers..."
for pod in $BLUE_PODS; do
    kubectl exec $pod -- curl -X POST http://localhost:8080/management/consumers/stop &
done
wait

# Step 2: Wait for in-flight messages to complete (configurable timeout)
echo "Waiting for in-flight messages to complete..."
sleep 30

# Step 3: Verify no active consumers in blue
for pod in $BLUE_PODS; do
    STATUS=$(kubectl exec $pod -- curl -s http://localhost:8080/management/consumers/status)
    echo "Blue pod $pod consumer status: $STATUS"
done

# Step 4: Start consumers in green environment
echo "Starting green consumers..."
for pod in $GREEN_PODS; do
    kubectl exec $pod -- curl -X POST http://localhost:8080/management/consumers/start &
done
wait

# Step 5: Verify consumers are active in green
for pod in $GREEN_PODS; do
    STATUS=$(kubectl exec $pod -- curl -s http://localhost:8080/management/consumers/status)
    echo "Green pod $pod consumer status: $STATUS"
done

echo "Consumer switch completed successfully"
```

#### 4.2 Monitor Consumer Group Reassignment

```bash
# Monitor Kafka consumer group reassignment
kubectl exec -it kafka-0 -- kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group payment-processor-blue

kubectl exec -it kafka-0 -- kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group payment-processor-green

# Monitor IBM MQ consumer activity
kubectl exec -it ibm-mq-0 -- echo "DISPLAY CONN(*)" | runmqsc QM1

# Check message processing metrics
kubectl logs -l slot=green --since=5m | grep "Message processed"
kubectl logs -l slot=blue --since=5m | grep "Message processed"
```

#### 4.3 Alternative: ConfigMap-Based Consumer Control

For more controlled switching, use ConfigMaps:

```yaml
# consumer-control-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: consumer-control
data:
  blue-active: "true"
  green-active: "false"
```

```yaml
# Update deployment to watch ConfigMap
spec:
  template:
    spec:
      containers:
      - name: message-processor
        env:
        - name: APP_CONSUMER_ACTIVE
          valueFrom:
            configMapKeyRef:
              name: consumer-control
              key: blue-active  # or green-active for green deployment
```

```bash
# Switch consumers by updating ConfigMap
kubectl patch configmap consumer-control \
  -p '{"data":{"blue-active":"false","green-active":"true"}}'

# Restart pods to pick up new configuration
kubectl rollout restart deployment/message-processor-blue
kubectl rollout restart deployment/message-processor-green
```

### Phase 5: Validation and Cleanup

#### 5.1 Post-Switch Monitoring

```bash
# Monitor message processing rates
kubectl exec -it prometheus-pod -- \
  promtool query instant 'rate(messages_processed_total{slot="green"}[5m])'

# Check for message processing errors
kubectl logs -l slot=green --since=10m | grep -i error

# Verify consumer lag (Kafka)
kubectl exec -it kafka-0 -- kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group payment-processor-green

# Monitor database transaction rates
kubectl exec -it cockroachdb-0 -- \
  /cockroach/cockroach sql --execute="SHOW CLUSTER QUERIES;"

# Check dead letter queue for failed messages
kubectl exec -it kafka-0 -- kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic payment-events-dlq \
  --from-beginning --max-messages 10
```

#### 5.2 Blue Environment Cleanup

```bash
# Wait for confidence period (typically 2-4 hours for message-driven apps)
sleep 7200

# Verify no messages are being processed by blue environment
BLUE_PROCESSING=$(kubectl logs -l slot=blue --since=1h | grep "Message processed" | wc -l)
if [ $BLUE_PROCESSING -eq 0 ]; then
    echo "Safe to scale down blue environment"
    
    # Scale down blue environment
    kubectl scale deployment message-processor-blue --replicas=0
    
    # Optional: Delete blue deployment after extended validation
    kubectl delete deployment message-processor-blue
else
    echo "Warning: Blue environment still processing messages"
fi
```

## Handling Database Migrations

### Migration Strategy for CockroachDB

#### 1. Backward-Compatible Migrations

```sql
-- Example: Adding a new column (backward compatible)
ALTER TABLE users ADD COLUMN email STRING DEFAULT '';

-- Example: Creating new index (backward compatible)
CREATE INDEX CONCURRENTLY idx_users_email ON users (email);
```

#### 2. Multi-Phase Migration Approach

```
Phase 1: Deploy code that can handle both old and new schema
Phase 2: Run schema migration
Phase 3: Deploy code that uses new schema exclusively
```

#### 3. Migration Script Example

```yaml
# migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-v1-1-0
spec:
  template:
    spec:
      containers:
      - name: migration
        image: myregistry/db-migrator:v1.1.0
        env:
        - name: DB_URL
          value: "postgresql://cockroachdb-service:26257/myapp"
        command:
        - /bin/sh
        - -c
        - |
          echo "Starting database migration..."
          /app/migrate -database "${DB_URL}" -path /migrations up
          echo "Migration completed successfully"
      restartPolicy: Never
  backoffLimit: 3
```

### Migration Execution

```bash
# Run migration before traffic switch
kubectl apply -f migration-job.yaml

# Monitor migration progress
kubectl logs job/db-migration-v1-1-0 -f

# Verify migration success
kubectl get job db-migration-v1-1-0 -o jsonpath='{.status.succeeded}'
```

## Message Queue Consumer Management

### IBM MQ Consumer Management

#### 1. Queue-Based Consumer Strategy

```java
@Component
public class IBMMQConsumerManager {
    
    @Autowired
    private ConnectionFactory connectionFactory;
    
    private List<Connection> activeConnections = new ArrayList<>();
    private volatile boolean consumersActive = false;
    
    @Value("${app.consumer.active:false}")
    private boolean shouldBeActive;
    
    @PostConstruct
    public void initialize() {
        if (shouldBeActive) {
            startConsumers();
        }
    }
    
    public void startConsumers() {
        if (consumersActive) return;
        
        try {
            // Create connections for each queue
            Connection paymentConnection = connectionFactory.createConnection();
            Connection notificationConnection = connectionFactory.createConnection();
            
            paymentConnection.start();
            notificationConnection.start();
            
            activeConnections.addAll(Arrays.asList(paymentConnection, notificationConnection));
            consumersActive = true;
            
            log.info("IBM MQ consumers started successfully");
        } catch (Exception e) {
            log.error("Failed to start IBM MQ consumers", e);
            throw new RuntimeException(e);
        }
    }
    
    public void stopConsumers() {
        if (!consumersActive) return;
        
        activeConnections.forEach(connection -> {
            try {
                connection.close();
            } catch (Exception e) {
                log.warn("Error closing MQ connection", e);
            }
        });
        
        activeConnections.clear();
        consumersActive = false;
        
        log.info("IBM MQ consumers stopped successfully");
    }
    
    @JmsListener(destination = "PAYMENT.QUEUE", 
                 condition = "#{@consumerLifecycleManager.isActive()}")
    public void processPaymentMessage(String message) {
        // Process payment message
        log.info("Processing payment message: {}", message);
    }
    
    @JmsListener(destination = "NOTIFICATION.QUEUE", 
                 condition = "#{@consumerLifecycleManager.isActive()}")
    public void processNotificationMessage(String message) {
        // Process notification message
        log.info("Processing notification message: {}", message);
    }
}
```

#### 2. IBM MQ Configuration for Blue-Green

```yaml
# Blue environment MQ configuration
spring:
  jms:
    ibm:
      mq:
        queue-manager: QM1
        channel: DEV.APP.SVRCONN
        conn-name: ibm-mq-service(1414)
        user: app-blue
        application-name: payment-processor-blue
        client-id: blue-client-${random.uuid}

# Green environment MQ configuration  
spring:
  jms:
    ibm:
      mq:
        queue-manager: QM1
        channel: DEV.APP.SVRCONN
        conn-name: ibm-mq-service(1414)
        user: app-green
        application-name: payment-processor-green
        client-id: green-client-${random.uuid}
```

### Kafka Consumer Management

#### 1. Consumer Group Strategy

```java
@Component
public class KafkaConsumerManager {
    
    @Autowired
    private KafkaListenerEndpointRegistry registry;
    
    @Value("${spring.kafka.consumer.group-id}")
    private String consumerGroupId;
    
    @Value("${app.consumer.active:false}")
    private boolean shouldBeActive;
    
    @PostConstruct
    public void initialize() {
        if (!shouldBeActive) {
            stopAllConsumers();
        }
    }
    
    public void startAllConsumers() {
        registry.getListenerContainers().forEach(container -> {
            if (!container.isRunning()) {
                container.start();
                log.info("Started Kafka consumer container: {}", container.getGroupId());
            }
        });
    }
    
    public void stopAllConsumers() {
        registry.getListenerContainers().forEach(container -> {
            if (container.isRunning()) {
                container.stop();
                log.info("Stopped Kafka consumer container: {}", container.getGroupId());
            }
        });
    }
    
    @KafkaListener(topics = "payment-events", 
                   groupId = "${spring.kafka.consumer.group-id}",
                   autoStartup = "${app.consumer.active:false}")
    public void processPaymentEvent(String message) {
        log.info("Processing payment event: {}", message);
        // Process payment event
    }
    
    @KafkaListener(topics = "user-events", 
                   groupId = "${spring.kafka.consumer.group-id}",
                   autoStartup = "${app.consumer.active:false}")
    public void processUserEvent(String message) {
        log.info("Processing user event: {}", message);
        // Process user event
    }
}
```

#### 2. Consumer Group Offset Management

```bash
# Before switching: Check current offsets for blue consumer group
kubectl exec -it kafka-0 -- kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --describe --group payment-processor-blue

# During switch: Reset green consumer group to start from current position
kubectl exec -it kafka-0 -- kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group payment-processor-green \
  --reset-offsets --to-current --topic payment-events --execute

# After switch: Monitor green consumer group lag
kubectl exec -it kafka-0 -- kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --describe --group payment-processor-green
```

### Consumer Coordination During Deployment

#### 1. Message Processing Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                Consumer Switching Process                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Phase 1: Blue Active, Green Inactive                          │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐   │
│  │             │    │              │    │                 │   │
│  │ Message     │────│ Blue Env     │────│   Database      │   │
│  │ Sources     │    │ (Processing) │    │   (Updates)     │   │
│  │             │    │              │    │                 │   │
│  └─────────────┘    └──────────────┘    └─────────────────┘   │
│                                                                 │
│  Phase 2: Consumer Switch (Critical Window)                    │
│  ┌─────────────┐                         ┌─────────────────┐   │
│  │             │    Stop Blue ───────────│                 │   │
│  │ Message     │         │               │   Database      │   │
│  │ Sources     │         ▼               │   (No Updates)  │   │
│  │             │    Start Green ─────────│                 │   │
│  └─────────────┘                         └─────────────────┘   │
│                                                                 │
│  Phase 3: Green Active, Blue Inactive                          │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐   │
│  │             │    │              │    │                 │   │
│  │ Message     │────│ Green Env    │────│   Database      │   │
│  │ Sources     │    │ (Processing) │    │   (Updates)     │   │
│  │             │    │              │    │                 │   │
│  └─────────────┘    └──────────────┘    └─────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

#### 2. Coordinated Consumer Lifecycle

```java
@Component
public class CoordinatedConsumerManager {
    
    @Autowired
    private KafkaConsumerManager kafkaManager;
    
    @Autowired
    private IBMMQConsumerManager mqManager;
    
    @EventListener
    public void handleConsumerActivation(ConsumerActivationEvent event) {
        if (event.isActivate()) {
            startAllConsumers();
        } else {
            stopAllConsumers();
        }
    }
    
    @Transactional
    public void startAllConsumers() {
        try {
            // Start Kafka consumers first (faster startup)
            kafkaManager.startAllConsumers();
            
            // Then start IBM MQ consumers
            mqManager.startConsumers();
            
            // Update consumer status in database
            updateConsumerStatus("ACTIVE");
            
            log.info("All consumers started successfully");
        } catch (Exception e) {
            log.error("Failed to start consumers, rolling back", e);
            stopAllConsumers();
            throw e;
        }
    }
    
    @Transactional
    public void stopAllConsumers() {
        try {
            // Stop IBM MQ consumers first (they may have longer processing times)
            mqManager.stopConsumers();
            
            // Then stop Kafka consumers
            kafkaManager.stopAllConsumers();
            
            // Update consumer status in database
            updateConsumerStatus("INACTIVE");
            
            log.info("All consumers stopped successfully");
        } catch (Exception e) {
            log.error("Error stopping consumers", e);
        }
    }
    
    private void updateConsumerStatus(String status) {
        // Update database to track consumer status
    }
}

## Alternative Deployment Strategies

### 1. Message Processing Deployment

**Pros:**
- Zero message loss
- Consumer group isolation
- Instant rollback capability
- No duplicate processing during switch

**Cons:**
- Brief processing gap during consumer switch
- Complex consumer management
- Requires careful offset management

**Implementation:**
```bash
# Message processing switch
./switch-consumers.sh blue green
```

### 2. Parallel Processing Deployment

**Pros:**
- No processing gap
- Gradual transition possible
- Both versions can run simultaneously

**Cons:**
- Potential duplicate processing
- Complex message deduplication
- Higher resource usage

**Implementation:**
```yaml
# Both environments process messages with deduplication
spec:
  strategy:
    type: ParallelProcessing
    deduplication: true
```

### 3. Queue Redirection Deployment

**Pros:**
- Message broker controls routing
- Fine-grained traffic control
- No application changes needed

**Cons:**
- Requires message broker features
- Complex queue configuration
- Potential message loss during routing changes

### Comparison Matrix for Message-Driven Applications

| Strategy | Message Loss Risk | Processing Gap | Complexity | Duplicate Processing | Rollback Speed |
|----------|------------------|----------------|------------|---------------------|----------------|
| Consumer Switch | None | Minimal (30s) | Medium | None | Instant |
| Parallel Processing | None | None | High | Possible | Fast |
| Queue Redirection | Low | None | Very High | None | Medium |
| Rolling Update | High | High | Low | Possible | Slow |

**Recommendation**: For message-driven applications, **Consumer Switch (Blue-Green)** is recommended due to its guarantee of no message loss and instant rollback capability.

## Best Practices

### 1. Infrastructure Best Practices

#### Resource Management
```yaml
# Resource quotas for blue-green environments
apiVersion: v1
kind: ResourceQuota
metadata:
  name: blue-green-quota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "10"
```

#### Network Policies
```yaml
# Network policy for environment isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: blue-green-isolation
spec:
  podSelector:
    matchLabels:
      app: spring-boot-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: load-balancer
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: cockroachdb
  - to:
    - podSelector:
        matchLabels:
          app: kafka
```

### 2. Application Best Practices

#### Health Checks Implementation for Message-Driven Apps
```java
@Component
public class MessageConsumerHealthIndicator implements HealthIndicator {
    
    @Autowired
    private KafkaConsumerManager kafkaConsumerManager;
    
    @Autowired
    private IBMMQConsumerManager mqConsumerManager;
    
    @Autowired
    private ConsumerLifecycleManager consumerLifecycleManager;
    
    @Override
    public Health health() {
        Health.Builder status = Health.up();
        
        // Check database connectivity
        if (!isDatabaseHealthy()) {
            status.down().withDetail("database", "Connection failed");
        }
        
        // Check Kafka consumer connectivity
        if (!isKafkaConsumerHealthy()) {
            status.down().withDetail("kafka-consumer", "Consumer group disconnected");
        }
        
        // Check IBM MQ consumer connectivity
        if (!isMQConsumerHealthy()) {
            status.down().withDetail("ibm-mq-consumer", "Queue consumer unavailable");
        }
        
        // Check consumer status
        boolean consumersActive = consumerLifecycleManager.areConsumersActive();
        status.withDetail("consumers-active", consumersActive);
        
        // Check message processing rate
        long recentMessageCount = getRecentMessageProcessingCount();
        status.withDetail("recent-message-count", recentMessageCount);
        
        return status.build();
    }
    
    private boolean isKafkaConsumerHealthy() {
        try {
            return kafkaConsumerManager.isHealthy();
        } catch (Exception e) {
            return false;
        }
    }
    
    private boolean isMQConsumerHealthy() {
        try {
            return mqConsumerManager.isHealthy();
        } catch (Exception e) {
            return false;
        }
    }
    
    private long getRecentMessageProcessingCount() {
        // Return count of messages processed in last 5 minutes
        return messageMetricsService.getRecentProcessingCount(Duration.ofMinutes(5));
    }
}
```

#### Consumer-Aware Graceful Shutdown
```java
@Component
public class MessageConsumerGracefulShutdown {
    
    @Autowired
    private ConsumerLifecycleManager consumerManager;
    
    @Value("${app.shutdown.consumer-drain-timeout:60}")
    private int consumerDrainTimeoutSeconds;
    
    @PreDestroy
    public void onDestroy() {
        log.info("Starting graceful shutdown for message consumers...");
        
        // Step 1: Stop accepting new messages
        consumerManager.stopConsumers();
        log.info("Stopped accepting new messages");
        
        // Step 2: Wait for in-flight messages to complete
        waitForInFlightMessages();
        
        // Step 3: Close connections gracefully
        closeConnections();
        
        log.info("Graceful shutdown completed");
    }
    
    private void waitForInFlightMessages() {
        int waited = 0;
        while (waited < consumerDrainTimeoutSeconds) {
            if (consumerManager.getInFlightMessageCount() == 0) {
                log.info("All in-flight messages processed");
                return;
            }
            
            try {
                Thread.sleep(1000);
                waited++;
                
                if (waited % 10 == 0) {
                    log.info("Still waiting for {} in-flight messages, waited {}s", 
                           consumerManager.getInFlightMessageCount(), waited);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
        
        log.warn("Timeout reached, forcing shutdown with {} in-flight messages", 
                consumerManager.getInFlightMessageCount());
    }
    
    private void closeConnections() {
        consumerManager.closeAllConnections();
    }
}

### 3. Monitoring and Observability

#### Metrics Collection for Message Processing
```yaml
# ServiceMonitor for Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: message-processor-metrics
spec:
  selector:
    matchLabels:
      app: message-processor
  endpoints:
  - port: metrics
    path: /actuator/prometheus
    interval: 15s  # More frequent for message processing
```

#### Custom Dashboards for Message-Driven Apps
```json
{
  "dashboard": {
    "title": "Message-Driven Blue-Green Deployment Dashboard",
    "panels": [
      {
        "title": "Message Processing Rate by Environment",
        "targets": [
          {
            "expr": "rate(messages_processed_total{app=\"message-processor\"}[5m])"
          }
        ]
      },
      {
        "title": "Consumer Lag by Environment",
        "targets": [
          {
            "expr": "kafka_consumer_lag_sum{group=~\".*-blue|.*-green\"}"
          }
        ]
      },
      {
        "title": "Message Processing Errors",
        "targets": [
          {
            "expr": "rate(message_processing_errors_total[5m])"
          }
        ]
      },
      {
        "title": "Dead Letter Queue Messages",
        "targets": [
          {
            "expr": "kafka_topic_partition_current_offset{topic=~\".*-dlq\"}"
          }
        ]
      },
      {
        "title": "Active Consumer Count",
        "targets": [
          {
            "expr": "consumer_active_count{app=\"message-processor\"}"
          }
        ]
      }
    ]
  }
}
```

### 4. Security Considerations

#### RBAC Configuration
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: blue-green-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "update", "patch"]
```

#### Secret Management
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  db-password: <base64-encoded-password>
  mq-password: <base64-encoded-password>
  kafka-cert: <base64-encoded-certificate>
```

## Monitoring and Rollback

### 1. Automated Monitoring Setup for Message Processing

#### Key Metrics to Monitor
- Message processing rate (messages/second per environment)
- Consumer lag (Kafka) and queue depth (IBM MQ)
- Message processing error rate (< 0.01% target)
- Dead letter queue message count
- Database transaction success rate
- Consumer connectivity status

#### Monitoring Script for Message-Driven Apps
```bash
#!/bin/bash
# message-processing-monitoring.sh

NAMESPACE="production"
GREEN_DEPLOYMENT="message-processor-green"
BLUE_DEPLOYMENT="message-processor-blue"

# Function to get message processing error rate
get_message_error_rate() {
    kubectl exec -it prometheus-pod -- \
        promtool query instant \
        'rate(message_processing_errors_total[5m]) / rate(messages_processed_total[5m]) * 100'
}

# Function to get consumer lag
get_consumer_lag() {
    kubectl exec -it kafka-0 -- kafka-consumer-groups \
        --bootstrap-server localhost:9092 \
        --describe --group payment-processor-green \
        | awk 'NR>1 {sum+=$5} END {print sum}'
}

# Function to get dead letter queue count
get_dlq_count() {
    kubectl exec -it kafka-0 -- kafka-run-class kafka.tools.GetOffsetShell \
        --broker-list localhost:9092 \
        --topic payment-events-dlq \
        | awk -F':' '{sum+=$3} END {print sum}'
}

# Function to check consumer connectivity
check_consumer_connectivity() {
    local pod=$(kubectl get pods -l slot=green -o jsonpath='{.items[0].metadata.name}')
    kubectl exec $pod -- curl -s http://localhost:8080/management/consumers/status | \
        jq -r '.kafkaConsumersActive and .mqConsumersActive'
}

# Monitoring loop
while true; do
    ERROR_RATE=$(get_message_error_rate)
    CONSUMER_LAG=$(get_consumer_lag)
    DLQ_COUNT=$(get_dlq_count)
    CONSUMER_STATUS=$(check_consumer_connectivity)
    
    echo "Message Error Rate: ${ERROR_RATE}%"
    echo "Consumer Lag: ${CONSUMER_LAG} messages"
    echo "Dead Letter Queue Count: ${DLQ_COUNT}"
    echo "Consumer Connectivity: ${CONSUMER_STATUS}"
    
    # Rollback conditions for message-driven apps
    if (( $(echo "$ERROR_RATE > 0.1" | bc -l) )); then
        echo "ERROR: Message error rate exceeded threshold, initiating rollback"
        ./message-consumer-rollback.sh
        break
    fi
    
    if (( CONSUMER_LAG > 10000 )); then
        echo "ERROR: Consumer lag too high, initiating rollback"
        ./message-consumer-rollback.sh
        break
    fi
    
    if [ "$CONSUMER_STATUS" != "true" ]; then
        echo "ERROR: Consumer connectivity issues, initiating rollback"
        ./message-consumer-rollback.sh
        break
    fi
    
    sleep 30
done
```

### 2. Automated Rollback Strategy for Message Consumers

#### Consumer Rollback Script
```bash
#!/bin/bash
# message-consumer-rollback.sh

echo "Initiating emergency consumer rollback..."

# Step 1: Stop green environment consumers immediately
GREEN_PODS=$(kubectl get pods -l slot=green -o jsonpath='{.items[*].metadata.name}')
echo "Stopping green consumers..."
for pod in $GREEN_PODS; do
    kubectl exec $pod -- curl -X POST http://localhost:8080/management/consumers/stop &
done
wait

# Step 2: Start blue environment consumers
BLUE_PODS=$(kubectl get pods -l slot=blue -o jsonpath='{.items[*].metadata.name}')
echo "Starting blue consumers..."
for pod in $BLUE_PODS; do
    kubectl exec $pod -- curl -X POST http://localhost:8080/management/consumers/start &
done
wait

# Step 3: Verify blue consumers are processing messages
sleep 10
for pod in $BLUE_PODS; do
    STATUS=$(kubectl exec $pod -- curl -s http://localhost:8080/management/consumers/status)
    echo "Blue pod $pod consumer status: $STATUS"
done

# Step 4: Check message processing resumption
echo "Waiting for message processing to resume..."
sleep 30

PROCESSING_COUNT=$(kubectl logs -l slot=blue --since=30s | grep "Processing.*message" | wc -l)
if [ $PROCESSING_COUNT -gt 0 ]; then
    echo "Message processing resumed successfully on blue environment"
else
    echo "WARNING: No message processing detected on blue environment"
fi

echo "Consumer rollback completed"

# Send notification
curl -X POST https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK \
    -H 'Content-type: application/json' \
    --data '{"text":"🚨 Emergency consumer rollback completed for message-processor"}'
```

#### Health Check Based Consumer Rollback
```yaml
# consumer-health-rollback-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: consumer-health-rollback
spec:
  template:
    spec:
      containers:
      - name: health-checker
        image: curlimages/curl:latest
        command:
        - /bin/sh
        - -c
        - |
          # Check consumer health for 5 minutes
          for i in {1..10}; do
            GREEN_POD=$(kubectl get pods -l slot=green -o jsonpath='{.items[0].metadata.name}')
            
            # Check consumer status
            CONSUMER_STATUS=$(kubectl exec $GREEN_POD -- \
              curl -s http://localhost:8080/management/consumers/status | \
              jq -r '.kafkaConsumersActive and .mqConsumersActive')
            
            # Check recent message processing
            RECENT_PROCESSING=$(kubectl logs -l slot=green --since=60s | \
              grep "Processing.*message" | wc -l)
            
            if [ "$CONSUMER_STATUS" != "true" ] || [ $RECENT_PROCESSING -eq 0 ]; then
              echo "Consumer health check failed, triggering rollback"
              ./message-consumer-rollback.sh
              exit 1
            fi
            
            sleep 30
          done
      restartPolicy: Never
```

### 3. Rollback Decision Matrix for Message-Driven Applications

| Metric | Threshold | Action | Priority | Recovery Time |
|--------|-----------|--------|----------|---------------|
| Message Error Rate | > 0.1% | Immediate Rollback | Critical | 2-3 minutes |
| Consumer Lag | > 10,000 messages | Investigate/Rollback | High | 5 minutes |
| Dead Letter Queue | > 100 messages | Investigate/Rollback | High | 10 minutes |
| Consumer Connectivity | Failure | Immediate Rollback | Critical | 1 minute |
| Database Transaction Failures | > 1% | Immediate Rollback | Critical | 2 minutes |
| Message Processing Rate | < 50% of normal | Investigate/Rollback | Medium | 15 minutes |

## Conclusion

Blue-green deployment for message-driven Spring Boot applications requires a fundamentally different approach than traditional HTTP-based services. Instead of routing traffic through load balancers, success depends on carefully orchestrated consumer management across IBM MQ and Apache Kafka.

### Key Takeaways for Message-Driven Applications

1. **Consumer Group Strategy**: Proper isolation of consumer groups between blue and green environments is critical
2. **Message Ordering**: Ensure message processing order is maintained during consumer switches
3. **Zero Message Loss**: Implement graceful consumer shutdown and startup procedures
4. **Monitoring Focus**: Monitor consumer lag, processing rates, and dead letter queues rather than HTTP metrics
5. **Rollback Speed**: Consumer switching enables faster rollback than traditional deployments

### Critical Success Factors

- **Idempotent Message Processing**: Design message handlers to be idempotent to handle potential duplicate processing
- **Dead Letter Queue Management**: Implement proper error handling and poison message detection
- **Consumer Health Monitoring**: Track consumer connectivity and processing rates continuously
- **Database Transaction Management**: Ensure database changes are compatible with both versions during transition
- **Configuration Management**: Use environment-specific consumer group IDs and connection settings

### Benefits for Message-Driven Architecture

1. **Zero Message Loss**: No messages are lost during deployment
2. **Instant Rollback**: Consumer switching provides immediate rollback capability
3. **Processing Isolation**: Blue and green environments never process the same messages simultaneously
4. **Reduced Risk**: Test message processing thoroughly before switching consumers
5. **Operational Control**: Fine-grained control over message consumption during deployments

### Next Steps

1. Implement consumer lifecycle management in your Spring Boot application
2. Set up consumer group monitoring and alerting
3. Create automated consumer switching scripts
4. Implement comprehensive message processing health checks
5. Practice consumer rollback scenarios regularly
6. Set up dead letter queue monitoring and alerting

### Operational Considerations

- **Consumer Lag Monitoring**: Set up alerts for high consumer lag across both environments
- **Message Replay Capability**: Implement ability to replay messages from specific offsets if needed
- **Cross-Environment Testing**: Test message flow from production queues to green environment safely
- **Consumer Group Cleanup**: Regularly clean up unused consumer groups and offsets

By following this guide and adapting the consumer-centric approach to your specific message-driven requirements, you can achieve reliable, zero-message-loss deployments for your mission-critical Spring Boot applications.

---

*This guide provides a comprehensive approach to blue-green deployment specifically designed for message-driven applications. The focus on consumer management rather than traffic routing makes it uniquely suitable for applications that process messages from IBM MQ and Apache Kafka without exposing HTTP business endpoints.*
