# CST8915 Lab 7: Introduction to Kubernetes Basics

| Student Name | Student ID | Course | Semester |
|---|---|---|---|
| Peng Wang | 041107730 | CST8915 Full-stack Cloud-native Development | Fall 2026 |

🎥 [Watch Demo Video](https://www.youtube.com/watch?v=UYC-Bl_p_q8)

---

## 1. Demo Video

### 1.1 Deploy the Application

```bash
kubectl apply -f algonquin-pet-store-all-in-one.yaml
```

### 1.2 Show the Pods

All four Pods reach `Running`.

```bash
kubectl get pods -o wide
```

### 1.3 Show the Services

The `EXTERNAL-IP` of `store-front` is the address used in the next steps.

```bash
kubectl get services
```

### 1.4 Open the Store Front

Open `http://172.210.126.135` in the browser.

### 1.5 Place Two Orders

On the Store Front, add one product to the cart and place the order. Repeat with a second product.

### 1.6 Check the Queue

`order_queue` is durable and holds 2 messages.

```bash
kubectl exec deployment/rabbitmq -- rabbitmqctl list_queues name durable messages
```

### 1.7 Show the Queue in RabbitMQ

Open `http://172.210.126.135/rabbitmq/#/queues` and log in with the credentials from the YAML file. `order_queue` shows 2 ready messages.

### 1.8 Delete the RabbitMQ Pod

```bash
kubectl delete pod -l app=rabbitmq
```

### 1.9 Show the New RabbitMQ Pod

The Deployment creates a replacement Pod with a new name.

```bash
kubectl get pods -l app=rabbitmq
```

### 1.10 Check the Queue Again

Wait about 10 seconds for the new node to start. `order_queue` and its 2 messages are gone.

```bash
kubectl exec deployment/rabbitmq -- rabbitmqctl list_queues name durable messages
```

Refresh the RabbitMQ page in the browser; it shows the same result.

## 2. RabbitMQ Configuration Analysis

### 2.1 Is RabbitMQ a Stateless or Stateful Application?

RabbitMQ is a stateful application. It stores queues, messages, users and its own node identity on disk under `/var/lib/rabbitmq`, and the orders in `order_queue` are data that must survive until a consumer processes them.

### 2.2 The Implications of Running RabbitMQ Without Persistent Storage

The RabbitMQ Deployment in `algonquin-pet-store-all-in-one.yaml` mounts no volume, so `/var/lib/rabbitmq` lives in the container's own filesystem. Kubernetes states that "On-disk files in a container are ephemeral" (Kubernetes, n.d.-b), so every order that is still waiting in the queue is stored only as long as that one container exists.

The order-service code already does what RabbitMQ asks for durability: it declares `order_queue` with `durable: true` and publishes with `persistent: true`. RabbitMQ recovers such messages "on node boot" (RabbitMQ, n.d.-d), but only from the disk the node finds when it starts. Without a persistent volume there is no disk to recover from, so these settings give no protection here.

The Deployment also runs a single replica. While that Pod is being replaced, there is no broker for order-service to publish to.

### 2.3 What Happens When the RabbitMQ Pod Is Deleted or Restarted

When the Pod is deleted, the Deployment creates a new Pod with a new name, an empty filesystem and a new hostname. A RabbitMQ node computes its name from the hostname when no name is configured (RabbitMQ, n.d.-a), so the new Pod also becomes a different node (`rabbit@<new pod name>`). The demo shows the result: `order_queue` and its 2 messages are gone.

When only the container restarts inside the same Pod, kubelet "restarts the container with a clean state" (Kubernetes, n.d.-b), so the data is lost in that case as well.

The loss is easy to miss, because the new Pod creates `myuser` again from the environment variables and order-service declares `order_queue` again on the next order. The store keeps working, but the orders that were waiting are silently lost.

### 2.4 Potential Solutions

| # | Solution | What it fixes | Trade-off |
|---|---|---|---|
| 1 | Run RabbitMQ as a StatefulSet with a `volumeClaimTemplate` that mounts an Azure Disk at `/var/lib/rabbitmq` | A StatefulSet gives "Stable, persistent storage" and "Stable, unique network identifiers" (Kubernetes, n.d.-a), so the Pod `rabbitmq-0` keeps both its disk and its node name after a restart. On AKS, a persistent volume "can exist beyond the lifetime of an individual pod" (Microsoft, n.d.-a). | Still one broker, so the queue is unavailable while the Pod restarts. |
| 2 | Use quorum queues on a three-node RabbitMQ cluster | Quorum queues are "replicated, durable, data-safety oriented" (RabbitMQ, n.d.-c), so losing one node does not lose messages. | Three Pods and three disks to run and pay for. |
| 3 | Deploy RabbitMQ with the RabbitMQ Cluster Kubernetes Operator | The Operator deploys RabbitMQ clusters and runs their "day two" operations (RabbitMQ, n.d.-b), and creates the StatefulSet and volumes for solutions 1 and 2. | One more component to install and learn. |
| 4 | Replace RabbitMQ with a managed broker such as Azure Service Bus | The broker and its storage are run by Azure, so there is no Pod or volume to manage. | Discussed in 2.5. |

### 2.5 Does Using Azure Service Bus Solve the Issues Identified with RabbitMQ Configuration in This Lab?

Yes for the issues found in this lab. Once Service Bus accepts a message, "it always holds the message durably in triple-redundant storage" (Microsoft, n.d.-b). Deleting or restarting a Pod in the cluster cannot remove the queue or its messages, because the broker no longer runs in the cluster.

It brings other costs. The primary protocol of Service Bus is AMQP 1.0 (Microsoft, n.d.-b), while order-service uses the RabbitMQ client library `amqplib`, so order-service and any consumer must be changed to use the Service Bus SDK and a connection string or managed identity. The application then depends on Azure and pays per use, and local development needs a Service Bus namespace or an emulator.

## 3. Notes

| # | Challenge | Fix |
|---|---|---|
| 1 | With one System node of Standard_D2as_v4 (2 vCPU), `product-service` and `store-front` stayed `Pending` with `Insufficient cpu`, because the AKS system Pods already requested 96% of the node's CPU. | Added a User node pool `workerspool` with one Standard_D2as_v4 node, which matches the "one worker node" in the handout and the Lab 6 layout. |
| 2 | Standard_D2as_v4 is available to this subscription only in East US. | Created the cluster in East US. |

## References

Kubernetes. (n.d.-a). *StatefulSets*. "Using StatefulSets", para. 1. https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#:~:text=Stable%2C%20persistent%20storage

Kubernetes. (n.d.-b). *Volumes*. "Why volumes are important", para. 1. https://kubernetes.io/docs/concepts/storage/volumes/#:~:text=On%2Ddisk%20files%20in%20a%20container%20are%20ephemeral

Microsoft. (n.d.-a). *Storage options for applications in Azure Kubernetes Service (AKS)*. "Persistent volumes", para. 1. Microsoft Learn. https://learn.microsoft.com/en-us/azure/aks/concepts-storage#:~:text=can%20exist%20beyond%20the%20lifetime%20of%20an%20individual%20pod

Microsoft. (n.d.-b). *What is Azure Service Bus?* "Queues", para. 2; "Compliance with standards and protocols", para. 1. Microsoft Learn. https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview#:~:text=it%20always%20holds%20the%20message%20durably%20in%20triple%2Dredundant%20storage

RabbitMQ. (n.d.-a). *Clustering guide*. "Node Names (Identifiers)", para. 4. https://www.rabbitmq.com/docs/clustering#:~:text=the%20node%20resolves%20its%20hostname%20and%20prepends

RabbitMQ. (n.d.-b). *RabbitMQ Kubernetes Operators*. "RabbitMQ Cluster Kubernetes Operator", para. 1. https://www.rabbitmq.com/kubernetes/operator/operator-overview#:~:text=including%20%22day%20two%22

RabbitMQ. (n.d.-c). *Persistence configuration*. "Overview of Persistence in RabbitMQ", list item 1. https://www.rabbitmq.com/docs/persistence-conf#:~:text=replicated%2C%20durable%2C%20data%2Dsafety%20oriented

RabbitMQ. (n.d.-d). *Queues*. "Durability", para. 3. https://www.rabbitmq.com/docs/queues#:~:text=Durable%20queues%20will%20be%20recovered%20on%20node%20boot

---

GenAI declaration: As permitted for labs in this course, Claude (Anthropic) was used to explain the steps, check portal settings and terminal output, and draft this README. The portal steps, commands and demo recording were done by the student.
