# Cloud native development with Dapr on GCP

## Resources
- [Cloud Pub/Sub](https://docs.dapr.io/reference/components-reference/supported-pubsub/setup-gcp-pubsub/)
- [Firestore](https://docs.dapr.io/reference/components-reference/supported-state-stores/setup-firestore/)


## Cloud Pub/Sub

### [Cloud Pub/Sub component config](https://docs.dapr.io/reference/components-reference/supported-pubsub/setup-gcp-pubsub/)

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: gcp-pubsub
spec:
  type: pubsub.gcp.pubsub
  version: v1
  metadata:
  - name: type
    value: service_account
  - name: projectId
    value: dascs-lab
  - name: identityProjectId
    value: dascs-lab
  - name: privateKeyId
    value: c0373af08aa9ae30d104841
  - name: clientEmail
    value: pubsub-admin-sa@lab.iam.gserviceaccount.com
  - name: clientId
    value: 1141644151051
  - name: authUri
    value: https://accounts.google.com/o/oauth2/auth
  - name: tokenUri
    value: https://oauth2.googleapis.com/token
  - name: authProviderX509CertUrl
    value: https://www.googleapis.com/oauth2/v1/certs
  - name: clientX509CertUrl
    value: https://www.googleapis.com/robot/v1/metadata/x509/lab.iam.gserviceaccount.com 
  - name: privateKey
    value: "-----BEGIN PRIVATE KEY-----\nMIIEvwIBADANBgkqhkiG9w0BAQEFAASCBKkwggSlAgEAAoIBAQDEQh...su0xJw==\n-----END PRIVATE KEY-----\n"
  - name: disableEntityManagement
    value: "false"
  - name: enableMessageOrdering
    value: "false"
  - name: maxReconnectionAttempts # Optional
    value: 30
  - name: connectionRecoveryInSec # Optional
    value: 2
```

### Java producer and consumer
- producer

```java
String TOPIC_NAME = "orders";
String PUBSUB_NAME = "gcp-pubsub";
DaprClient client = new DaprClientBuilder().build();

// Message Time-to-Live (TTL)
Map<String, String> metadata = new HashMap<>();
metadata.put("ttlInSeconds", "120");

client.publishEvent(PUBSUB_NAME,
        TOPIC_NAME,
        order, metadata).block();

logger.info("Published data: " + order.getOrderId());
```

- consumer

```java
@Topic(name = "orders", pubsubName = "gcp-pubsub")
@PostMapping(path = "/order-processor", consumes = MediaType.ALL_VALUE)
public Mono<ResponseEntity> getCheckout(@RequestBody(required = false) CloudEvent<Order> cloudEvent) {
        return Mono.fromSupplier(() -> {
            try {
                logger.info("Subscriber received: " + cloudEvent.getData().getOrderId());
                return ResponseEntity.ok("SUCCESS");
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });
    }
}
```

### [Declarative and programmatic subscription methods](https://docs.dapr.io/developing-applications/building-blocks/pubsub/subscription-methods/)

### Declarative subscriptions
You can subscribe declaratively to a topic using an external component file. This example uses a YAML component file named subscription.yaml:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: order
spec:
  topic: orders
  route: /order-processor
  pubsubname: gcp-pubsub
scopes:
- order-processor
```

Here the subscription called order:

- Uses the pub/sub component called **gcp-pubsub** to subscribes to the topic called orders.
- Sets the route field to send all topic messages to the **/order-processor** endpoint in the app.
- Sets scopes field to scope this subscription for access only by apps with IDs orderprocessing and checkout.

### In your application code, subscribe to the topic specified in the Dapr pub/sub component.

- The **/order-processor** endpoint matches the route defined in the subscriptions and this is where Dapr sends all topic messages to.

```java
@PostMapping(path = "/order-processor", consumes = MediaType.ALL_VALUE)
    public Mono<ResponseEntity> process(@RequestBody(required = false) CloudEvent<Order> cloudEvent) {
        return Mono.fromSupplier(() -> {
            try {
                logger.info("Subscriber received: " + cloudEvent.getData().getOrderId());
                return ResponseEntity.ok("SUCCESS");
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });
    }
}
```


