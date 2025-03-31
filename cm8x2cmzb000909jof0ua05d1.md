---
title: "Master Kubernetes Pod Scheduling: Taints, Tolerations, NodeSelector & Affinity Explained"
datePublished: Mon Mar 31 2025 12:46:41 GMT+0000 (Coordinated Universal Time)
cuid: cm8x2cmzb000909jof0ua05d1
slug: master-kubernetes-pod-scheduling-taints-tolerations-nodeselector-and-affinity-explained
tags: kubernetes, devops, nodeaffinity, taints-and-toleration, pod-scheduling

---

Being the K8s user, we all know that kube-scheduler is the component in the control plane that is responsible for scheduling the pods/workloads in the nodes inside kubernetes cluster. By default, the kube-scheduler checks the resource availability in the worker nodes and schedules the pods on any of the qualified node.

But, sometimes we need to schedule the pods on the specific nodes based on our requirement like running the ML workloads on the GPU-based nodes, database pods on the high IOPS supported disk, etc. and many more. So, there might be the situation where we need to:

i. Restrict the pods to be scheduled on certain nodes.

ii. Force the pods to run on specific nodes.

iii. Effectively place the pods on different nodes to ensure high availability.

No Worries. K8s provides the feature to achieve our requirements through Taints and Tolerations, NodeSelectors and Affinity.

### 1\. Taints & Tolerations: Prevent Pods from Running on Certain Nodes

This is useful when we want to restrict the pods to be scheduled on the certain nodes unless they have the matching toleration. You might have observed that the pods never get scheduled on the control plane nodes because the K8s control nodes are by default tainted so that the regular workloads don’t get scheduled.

At first, we need to taint the nodes as:

`kubectl taint node <node-name> <key>=<value>:<taint-effect>`

*The available taint-effects are:*

i. **NoSchedule** : It restricts the pod to get scheduled on the tainted nodes unless it has the matching tolerations. So, the pod will remain in pending state if any matching node is not found.

ii. **PreferNoSchedule**: It tries not to schedule the pod on the tainted node. But, if any of the matching nodes are not available, the pod gets scheduled on any of the node. So, the pod won’t be in the pending state.

iii. **NoExecute**: It terminates the pods running on the tainted node immediately if the pods don’t have any matching tolerations unlike the other two which doesn’t check the taints for the running pods.

Here’s the practical example of using the taints and tolerations.I have created the deployment that creates 3 replicas of `busybox` pods.

<mark>Deployment.yaml file</mark>

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: pod-scheduler

spec:
  replicas: 3
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels: 
        app: test
    spec:      
      containers:
        - name: busybox
          image: busybox
          resources:
            limits:
              cpu: "100m"
              memory: "64Mi"
          command: ["sleep", "3600"]
```

When I apply this deployment without any conditions then the `kube-scheduler` distributes the pods on all the nodes i.e.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1742979511498/021c4197-6af5-48d8-85ce-bed6b56b3df6.png align="center")

Now, I will taint the `minikube-m03` node and let’s see the result what happens.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1742980116417/c9c945d1-c051-4a9b-919c-d48d2f93e279.png align="center")

The taint effect `NoExecute` on the node `minikube-m03` removed the running pod on that node cause pod hasn’t any matching toleration for that taint and the new pod was created on the other node because the deployment controller ensures that the desired state always match will the actual state of the cluster.

Now, if I add the following toleration in the `spec.template.spec`.. section of the above deployment, then the pods with this toleration will be allowed to be scheduled on the tainted node i.e. `minikube-m03` but it’s not guaranteed that those pods will be scheduled on that tainted node.

```yaml
tolerations:
  - key: gpu
    value: "true"
    effect: NoExecute
    operator: Equal
```

In order to remove the taint on the node, the command will be:

```bash
kubectl taint node <node-name> <key>=<value>:<taint-effect>-
```

Only change will be adding hyphen(-) at the end. Keep in the mind that, `NoExecute` taint effect is for checking the running pods.

Now, what if we want the pods to be scheduled on the specific node or the group of nodes ? Here comes the `nodeSelector` and `Node Affinity` in action.

### 2\. NodeSelector & Node Affinity: Which One to Use?

**<mark>Node Selectors</mark>**

Node Selectors and affinity are the ways to force the pod to be scheduled on the particular matching node only based on the node labels. `nodeSelector` is the simple way to do this. For this, at first, we need to label the nodes with key-value pair as:

```bash
kubectl label node <node-name> <key>=<value>
```

I have also updated deployment in the `spec.template.spec`.. section with:

```yaml
nodeSelector:
    performance: low
```

After making the changes and applying the deployment config file. Here’s what happens.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1742984637556/1cd7b32a-66ed-4d1f-888d-9ae03cb0a035.png align="center")

All the pods are scheduled on that labelled node i.e. `minikube-m02`. But, if the `nodeSelector` label doesn’t get matched with the label of any nodes, then the pod will remain in the pending state. Let’s see the pending state as well by removing the label from the node (`minikube-m02`) as ( adding hypen (-) at the end ):

```bash
kubectl label node minikube-m02 performance-
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1743417953938/5fd7714e-93be-4d02-ba78-5a369e4d91cd.png align="center")

We can see that the pods are in pending state. So, `nodeSelector` is strict in nature. But, sometimes we want more flexibility like as using the taint effect `PreferNoSchedule`. Yup, we have `Node Affinity` that provide us this flexibility. Let’s have a quick look at it.

**<mark>Node Affinity</mark>**

There are two types of node affinity that work same as the taint effects in taints and tolerations.

i. ***requiredDuringSchedulingIgnoredDuringExecution***: It’s a strict type means there should be matching node labels for the pod to be scheduled otherwise the pod remains in pending state forever.

ii. ***preferredDuringSchedulingIgnoredDuringExecution***: It tries to schedule the pod in the node that has matching labels. But, if no node exists with that label/s then the pod will be scheduled in any of the nodes based on the default algorithm of `kube-scheduler`.

Are you also thinking about the suffix of the above effects `IgnoredDuringExecution` ? It means that these rules apply only at scheduling time. The phrase says it all , node affinity doesn’t have control over the already running pods and any changes to node labels will not affect already scheduled pods. They will be in the running state as they were earlier. Let’s have a practical demonstration.

Already, we have the previous label on `minikube-m02` as:

```bash
kubectl label node minikube-m02 performance=low
```

Let’s add the following affinity to the `spec.template.spec`.. section of the `deployment.yml` file. And, remove the previous `nodeSelector` field in `spec.template.spec`.. section.

```yaml
affinity:
    nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: performance
                    operator: In
                    values:
                      - low
```

We can see we have more flexibility while matching the labels. Multiple key-value pairs can be used, multiple values for a single key can also be used. Additionally, different operators like ( `In`, `Exists`, `NotIn`, `DoesNotExist` ), etc. can also be used. Check out more about `operators` here:

[https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#operators](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#operators)

See all the pods are scheduled in the `minikube-m02`. It works same as `nodeSelector` but with more flexibility.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1743420407250/c9da538a-11b6-4ba6-889e-30869ff01ebf.png align="center")

Now, let’s have a look at the `PreferNoSchedule` affinity.

Add this section in the `spec.template.spec`… section of the `deployment.yaml` file and remove the node-label of `minikube-m02`.

```yaml
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 5
            preference:
              matchExpressions:
              - key: performance
                operator: In
                values:
                - low
```

**Importance of weight field**

The `weight` field acts as a priority value assigned to a node when it matches the specified preference. Each matching expression can have a different weight. If multiple nodes satisfy the expression, then the `kube-scheduler` sums the weights of all matching expressions for each node and selects the node with the highest total weight for scheduling the pod. For example, consider two matching expressions:

* **Expression\_1** with a weight of **10**
    
* **Expression\_2** with a weight of **20**
    

Now, assume:

* **Node\_1** satisfies both **Expression\_1** and **Expression\_2**
    
* **Node\_2** satisfies only **Expression\_2**
    

The total weights would be:

* **Node\_1**: `10 (Expression_1) + 20 (Expression_2) = 30`
    
* **Node\_2**: `20 (Expression_2) = 20`
    

So, the scheduler schedules the pod in **Node\_1.**

  
Let’s see the cluster status after deploying the updated `deployment.yml` file:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1743421744360/073d9e89-9614-4294-8834-fd782d3621d0.png align="center")

The pods didn’t have the matching labels with any of the nodes. So they are scheduled by the default algorithm of `kube-scheduler` on the nodes. If we have used the `requiredDuringSchedulingIgnoredDuringExecution` instead of `preferredDuringSchedulingIgnoredDuringExecution` then all the pods will go in pending state*.*

If I wrap the things up then I would say `nodeAffinity` provides more flexible control over pod scheduling but is more complex to use. For strict matching of labels, it’s better to use `nodeSelector` or even the `nodeName` field that directly schedules the pod in the specific single node with the matching name.

Check out more on `nodeAffinity` here:

[https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)

Check out more on `nodeName` here:

[https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodename](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodename)

### Key Points to Remember:

i. `Taints` are used to restrict the pods to be scheduled on the nodes and `tolerations` helps the pod to overcome the matching taint and get schedule on that node.

ii. Even if the pod has matching toleration, then it’s not guaranteed that the pod will be scheduled on the tainted node.

iii. `nodeSelector`, `nodeAffinity` and `nodeName` are the ways to force the pod to be scheduled on the particular node.

If you want to go deeper into the pod scheduling on Kubernetes, then check this link below:

[https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)

Connect with me on:  
🔗 **LinkedIn**: [linkedin.com/in/anjal-poudel-8053a62b8](https://linkedin.com/in/anjal-poudel-8053a62b8)