# Lab 7

## Demo Video

[Youtube Link](https://youtu.be/KTlrlOF1WQw)

## Rabbit MQ: Configuration Analysis

### 1. Whether RabbitMQ is a stateless or stateful application

In the demo, when I scaled `rabbitmq` up to 2 pods using the command `kubectl scale deployment rabbitmq --replicas=2`, RabbitMQ Management threw a `502 Bad Gateway` error because the service now points to a new pod that does not have the expected state or queue for the two orders I had placed up until that point. This proves that our owned RabbitMQ service is stateful, where the queue and its messages exist within our first pod's memory, but not within any new pods spun up after the orders had been placed. Each pod is isolated from each other, so the management UI breaks once it fails to reconcile a consistent state between the pods.

### 2. The implications of running RabbitMQ without persistent storage

If we don't have a persistent storage set up with something like `PersistentVolumes`, the pods will not be able to reconcile their different queue/message state. In addition, when unhealthy pods are restarted by the orchestrator, the ephemeral nature of the containers within will result in lost local data, including queue data and schema. This means that messages and queues will disappear before any subscribed services can fulfill their processes, leading to potential business downtime in a production environment.

### 3. What happens when the RabbitMQ pod is deleted or restarted

When you delete a pod, AKS will automatically spin up a new one to match how many pod replicas you are currently scaled to in your node. For example, if you started with one RabbitMQ pod A and scale up to 2 pods, AKS will spin up pod B. If you delete pod A, AKS will spin up a new pod C. Because pods and its containers are ephemeral by nature, when you restart or delete a pod, you also wipe the in-memory state of the pod for all the containers in it. AKS will destroy the original pod entirely and rebuild it from the `spec` values defined in the YAML file. Any message queues stored in memory or in container filesystems in RabbitMQ pod A will be lost without any persistent storage to preserve state outside the pod.

### 4. Potential solutions to this problem (research-based)

One potential solution to the persistency issue is to use StatefulSets instead of Deployments in the YAML file. "A StatefulSet runs a group of Pods, and maintains a sticky identity for each of those Pods. This is useful for managing applications that need persistent storage or a stable, unique network identity." ([ref](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)) Each pod created from a StatefulSet have "persistent Pod identifiers [that] make it easier to match existing volumes to the new Pods that replace any that have failed." Therefore, while Deployments define how to "keep RabbitMQ pods running," `StatefulSets` with `PersistentVolumes` would define how to "keep RabbitMQ pods running and retain data between Pod restarts." ([ref](https://www.rabbitmq.com/blog/2020/08/10/deploying-rabbitmq-to-kubernetes-whats-involved)) Using the prior example, now when pod A restarts, the persistent Pod identifiers will allow AKS to rebuild the pod by provisioning with the `PersistentVolume` attached to each pod. ([ref](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)) Currently, when pod A restarts, pod A is completely destroyed and then rebuilt from the `spec` values. All the memory containing the queue and message data is lost. Furthermore, StatefulSets "ensures that the RabbitMQ nodes are deployed in order, one at a time."

A second solution that would align more with best cloud-native practices is to offload the responsibility through a managed backing message queue system instead of the current owned RabbitMQ service. The most compatible option here would be Azure Service Bus since it would naturally integrate within the Azure ecosystem with AKS. Since it is fully managed, they are designed to be scalable, highly available, and handle the persistence issues.

### 5. Does using Azure Service Bus solve the issues identified with RabbitMQ Configuration in this Lab?

Azure Service Bus would solve the issues of the owned RabbitMQ backing service. In this lab, we are responsible to manage RabbitMQ pods, scaling, and deal with the persistence challenges as a result of that management oversight. This leads to state loss on pod deletion/restart, desynchronized states between replicas, and ultimately manual setup for persistence. (PersistentVolume, StatefulSets) Meanwhile, Azure Service Bus comes with built-in persistence and high availability within Azure's infrastructure, where messages are "always held durably in triple-redundant storage, spread across availability zones if the namespace is zone-enabled. Service Bus keeps messages in memory or volatile storage until client reports them as accepted." ([ref](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview)) Azure handles all the scaling as well, freeing us from the pod management we are responsible with RabbitMQ.