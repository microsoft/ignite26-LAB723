# LAB723: Build, observe, and scale AI-ready apps and workloads with AKS

Deploy and operate an AI inference service on Azure Kubernetes Service (AKS) without spending the lab building a cluster. In this lab, you deploy a prepared inference service to AKS Automatic, work with the platform's built-in guardrails, tune the workload's resource profile, add GPU acceleration, watch the service scale under load, and run a distributed batch job with Ray on the same cluster.

**Level:** 200, intermediate  
**Duration:** 75 minutes

## What you'll learn

After completing this lab, you'll be able to:

- Deploy an AI inference application to AKS Automatic and make a noncompliant manifest comply with the platform's guardrails.
- Explain how AKS Automatic provisions compute from workload requirements, including GPU compute.
- Configure and compare resource profiles for an AI workload.
- Use metrics to evaluate latency, throughput, resource consumption, and workload health.
- Apply autoscaling and interpret pod and node scaling under load.
- Relate workload requirements to capacity, availability, and cost decisions.
- Describe where a distributed framework such as Ray fits in an AI architecture on AKS.

## Your lab environment

In the hosted lab, everything in the following table is already provisioned for you. You don't need your own Azure subscription, GPU quota, or local tools. To build a similar environment yourself, see [The lab environment](../docs/lab-environment.md).

| Component | What's ready for you |
|:--|:--|
| Lab VM | Visual Studio Code, Microsoft Edge, Azure CLI, `kubectl`, and `kubelogin`, configured for your environment |
| AKS Automatic cluster | Node autoprovisioning, deployment safeguards, workload identity, managed Prometheus, Container insights, and dashboards with Grafana |
| Registry and storage | Prepared inference images, and a compact model that the app reads from Azure Blob Storage by using workload identity |
| `ai-lab` namespace | **lab-client** (benchmark tool) and **loadgen** (load generator, scaled to zero) |
| `ray` namespace | A running Ray cluster (one head and several CPU workers) and a JupyterLab notebook |
| GPU capacity | Reserved for the accelerated profile in Exercise 3 |

## Lab timing

| Exercise | Time |
|:--|--:|
| [1. Orient and verify the environment](#exercise-1-orient-and-verify) | 5 min |
| [2. Deploy AI inference with AKS Automatic](#exercise-2-deploy-ai-inference) | 15 min |
| [3. Optimize AI inference on AKS](#exercise-3-optimize-ai-inference) | 20 min |
| [4. Observe, operate, and scale the workload](#exercise-4-observe-and-scale) | 15 min |
| [5. Run a distributed AI task](#exercise-5-run-a-distributed-ai-task) | 10 min |
| [6. Review and next steps](#exercise-6-review-and-next-steps) | 5 min |

## How to use these instructions

- In the hosted lab, these steps are also in the lab's instruction pane. There, you select a command to type it into the terminal, select the checkbox next to each step to track your progress, and type your measurements in text boxes that later steps reuse.
- On this page, copy each command into the terminal, and press **Enter** to run it. Write down the measurements that a step asks you to record, because Exercise 3 compares them.
- Run the commands from the folder that has the lab files: `C:\LabFiles\LAB723` in the hosted lab, which VS Code opens, or the `labfiles` folder of the [ai-lab repository](https://aka.ms/ignite26-ai-lab) on your own.
- On your own, set up an environment like the hosted lab's first, as [The lab environment](../docs/lab-environment.md) describes, and replace the placeholders in angle brackets in the lab files, such as `<registry>`, with your own values. Where a step uses the hosted lab's sign-in or names, use your own.
- Commands that end with `--watch` keep running until you press **Ctrl+C**.
- If you fall behind, look for the **Stuck?** section at the end of an exercise to catch up.

> [!NOTE]
> You need basic familiarity with containers and Kubernetes, and you should be comfortable reading short YAML manifests. No Python, Ray, or AI/ML experience is required.

## Exercise 1: Orient and verify

**Estimated time:** 5 minutes

Confirm that you can reach your environment, and review what's already running. If any step in this exercise fails, ask a proctor right away. They can move you to a spare environment.

### Task 1: Sign in

1. If the lab VM shows a sign-in screen, sign in with the password on the lab's **Resources** tab.

1. Open **Microsoft Edge**, go to `https://portal.azure.com`, and sign in with the following credentials:

    - **Username:** the username on the lab's **Resources** tab
    - **Temporary Access Pass:** the Temporary Access Pass on the lab's **Resources** tab

    Leave this tab open. You use the Azure portal again in Exercises 3 and 4.

### Task 2: Open the lab workspace

1. Open **Visual Studio Code** from the taskbar. If the **LAB723** folder isn't already open, select **File** > **Open Folder**, enter `C:\LabFiles\LAB723`, and open the folder.

1. Select **Terminal** > **New Terminal**. Unless a step says otherwise, run every command in this lab from this terminal.

### Task 3: Verify cluster access

1. Sign in to the Azure CLI with your lab account. Run the following command:

    ```shell
    az login
    ```

    A browser tab opens. Select your lab account, which you already signed in with in Task 1. If you're asked to sign in again, use the same Temporary Access Pass. When the tab says that you're signed in, return to Visual Studio Code.

    > **Note:** `kubectl` uses this sign-in. AKS Automatic clusters use Microsoft Entra ID for authentication and Azure RBAC for authorization, so you reach your cluster as your own lab account.

1. Run the following command to list the cluster's nodes and their VM sizes:

    ```shell
    kubectl get nodes -L node.kubernetes.io/instance-type
    ```

    Every node should show the status **Ready**.

    <details>
    <summary>Expand this section if kubectl asks you to sign in or returns an authorization error.</summary>

    Run `az login` again, and select your lab account, not another account. Then run the previous command again. If kubectl still can't reach the cluster, run the following commands in order, replacing `<resource-group>` with your lab's resource group, and then run the previous command again:

    ```shell
    az aks get-credentials --resource-group <resource-group> --name lab723-aks --overwrite-existing
    ```

    ```shell
    kubelogin convert-kubeconfig -l azurecli
    ```

    </details>

### Task 4: Review the starting state

1. Run the following command to list the node pools that node autoprovisioning (NAP) can use:

    ```shell
    kubectl get nodepools
    ```

    > **Tip:** In AKS Automatic, you don't size node pools yourself. NAP reads the resource requests of pods that can't be scheduled and creates a right-sized VM for them. Each `NodePool` limits which VM sizes NAP can choose. In this lab, the **default** pool allows general-purpose VM sizes, and the **gpu** pool allows only the GPU VM size reserved for this event, so NAP never picks a size that has no quota. You might also see **system-surge**, which NAP uses for system add-ons.

1. Run the following command to see what's already running in the lab namespace:

    ```shell
    kubectl get deployments,pods -n ai-lab
    ```

    You should see **lab-client** with one ready pod and **loadgen** with zero replicas. The inference service doesn't exist yet. You deploy it in the next exercise.

1. Run the following command to confirm that the Ray cluster you use in Exercise 5 is ready:

    ```shell
    kubectl get rayclusters -n ray
    ```

    The **STATUS** column should show **ready**.

> [!TIP]
> **Lab architecture.** Requests flow from the **lab-client** and **loadgen** pods to the **inference** service in the `ai-lab` namespace. The inference pods read the model from Azure Blob Storage by using workload identity, so no secrets are stored in the cluster. Managed Prometheus collects platform and application metrics, and dashboards with Grafana display them in the Azure portal. The Ray cluster in the `ray` namespace shares the same cluster and the same NAP-managed capacity.

## Exercise 2: Deploy AI inference

**Estimated time:** 15 minutes

Deploy the prepared inference service, see how the AKS Automatic guardrails respond to manifests that don't follow best practices, and record a baseline that you compare against in Exercise 3.

### Task 1: Review the prepared manifests

1. In VS Code, open **02-deploy/inference-deployment.yaml**, and review the settings called out in the following excerpt:

    ```yaml
    spec:
      selector:
        matchLabels:
          app: inference
      template:
        metadata:
          labels:
            app: inference
            azure.workload.identity/use: "true"   # inject workload identity credentials
        spec:
          serviceAccountName: inference           # federated with a managed identity
          topologySpreadConstraints:              # declared explicitly (see Task 3)
            - maxSkew: 1
              topologyKey: kubernetes.io/hostname
              whenUnsatisfiable: ScheduleAnyway
              labelSelector:
                matchLabels:
                  app: inference
          containers:
            - name: inference
              image: <registry>/lab723/inference:latest
              ports:
                - name: http
                  containerPort: 8080
              envFrom:
                - configMapRef:
                    name: lab-environment         # the model's storage account and container
                - secretRef:
                    name: appinsights             # the Application Insights connection string
                    optional: true
              env:
                - name: OMP_NUM_THREADS           # runtime threads match the CPU limit
                  value: "1"
                - name: INFERENCE_THREADS
                  value: "1"
              resources:                          # request = limit: Guaranteed QoS
                requests: { cpu: "1", memory: 2Gi }
                limits:   { cpu: "1", memory: 2Gi }
              readinessProbe:
                httpGet: { path: /ready, port: http }
                timeoutSeconds: 5
              livenessProbe:
                httpGet: { path: /healthz, port: http }
                timeoutSeconds: 5
                failureThreshold: 6
    ```

    In your copy of the file, `<registry>` is your lab's container registry, a name that ends in `.azurecr.io`.

    > **Tip:** Each setting answers a production question. **Requests and limits** reserve capacity and tell NAP how large a node to create. Setting the request equal to the limit gives the pod the *Guaranteed* quality-of-service class, which keeps latency measurements comparable. **Probes** keep traffic away from a pod until its model is loaded, and restart the container if it stops responding. **Workload identity** lets the app read its model from Azure Blob Storage with a managed identity instead of a stored secret. The **thread settings** keep the inference runtime from starting more threads than its CPU limit can run.

1. Run the following command to see the managed identity that the service account is federated with:

    ```shell
    kubectl describe serviceaccount inference -n ai-lab
    ```

    The **azure.workload.identity/client-id** annotation identifies the managed identity that has read access to the model's storage account.

### Task 2: See a guardrail that denies

1. Run the following command to apply the deployment, service, and ingress in the **02-deploy** folder:

    ```shell
    kubectl apply -f 02-deploy/
    ```

    The service and ingress are created, but the deployment is denied with an error similar to the following example:

    ```text
    Error from server (Forbidden): error when creating "02-deploy/inference-deployment.yaml":
    admission webhook "validation.gatekeeper.sh" denied the request: [azurepolicy-...]
    Please specify an explicit, versioned image tag such as '1.0' for container ...
    ```

1. Read the error message before you change anything. Deployment safeguards run in **Enforce** mode on AKS Automatic, so the platform rejects images tagged `latest`, which can change without notice.

1. In **02-deploy/inference-deployment.yaml**, select `latest` at the end of the `image:` line, replace it with `1.0.0`, and save the file.

1. Run the following command to apply the corrected deployment:

    ```shell
    kubectl apply -f 02-deploy/inference-deployment.yaml
    ```

    The output should read **deployment.apps/inference created**.

### Task 3: See a guardrail that mutates

Some safeguards don't block a workload. Instead, they change it.

1. In VS Code, open **02-deploy/guardrails/no-resources.yaml**. This small deployment, named **guardrail-demo**, has probes and a versioned image tag, but it has no resource requests or limits and no spreading rules.

1. Run the following command to apply it:

    ```shell
    kubectl apply -f 02-deploy/guardrails/no-resources.yaml
    ```

    The deployment is accepted.

1. Run the following command to view the resources of the pod that was created:

    ```shell
    kubectl get pods -n ai-lab -l app=guardrail-demo -o jsonpath='{.items[0].spec.containers[0].resources}'
    ```

    The pod has CPU and memory requests and limits of **500m** and **2Gi** that you never wrote. The resource requests mutator added them. If the command reports that the index is out of bounds, wait a few seconds, and run it again.

1. Run the following command to view the pod's topology spread constraints:

    ```shell
    kubectl get pods -n ai-lab -l app=guardrail-demo -o jsonpath='{.items[0].spec.topologySpreadConstraints}'
    ```

    The anti-affinity and topology spread mutator added a rule that spreads replicas across nodes. The inference deployment declares its own rule, so the mutator leaves it unchanged.

1. Run the following command to remove the demo deployment:

    ```shell
    kubectl delete -f 02-deploy/guardrails/no-resources.yaml
    ```

> [!TIP]
> Mutation is helpful and surprising at the same time: the deployment succeeds, but it doesn't run with the manifest that you wrote. A default of 500m CPU and 2Gi of memory might be far too much or too little for your workload, and it changes how NAP sizes nodes. Declare requests, limits, and spreading rules explicitly so that your manifest stays the source of truth.

### Task 4: Verify the rollout, routing, and logs

1. Run the following command, and wait until the rollout finishes:

    ```shell
    kubectl rollout status deployment/inference -n ai-lab --timeout=180s
    ```

    The pod is ready only after its readiness probe confirms that the model is loaded, which can take up to a minute.

1. Run the following command to review the deployment, pod, service, and ingress:

    ```shell
    kubectl get deployment,pods,service,ingress -n ai-lab
    ```

1. Run the following command to check the application logs:

    ```shell
    kubectl logs deployment/inference -n ai-lab --tail 20
    ```

    Look for the log entry that confirms the model was loaded from Azure Blob Storage by using workload identity.

1. In the output of the `kubectl get deployment,pods,service,ingress` command, find the **ADDRESS** of the **inference** ingress. If it's empty, wait a minute, and run `kubectl get ingress -n ai-lab`.

1. In Edge, open a new tab, and go to `http://<ADDRESS>/predict?text=This+lab+is+great`, replacing `<ADDRESS>` with the ingress address.

    The response is a prediction that came through the ingress: the sentiment **label** and **score** for the text, the **model**, the **device** that served the request, which is **cpu** for this deployment, and the **pod** that answered.

    > **Note:** The application routing add-on in AKS Automatic provides a managed NGINX ingress controller, so the ingress gets an address without any extra installation. In production, you'd typically put an API gateway in front of an AI service to add authentication, rate limiting, and cost controls.

    > **Tip:** The container's liveness probe uses `/healthz`, but `http://<ADDRESS>/healthz` shows an empty page. For an ingress without a host name, like this one, the NGINX controller answers `/healthz` itself, because cloud load balancers use that path to check the controller. The kubelet calls the probe on the pod directly, so the probe still works. When you publish a service through an ingress, avoid the paths that the controller reserves.

### Task 5: Run a first inference request and record a baseline

1. Run the following command to send one request to the inference service from the in-cluster lab client:

    ```shell
    kubectl exec -n ai-lab deploy/lab-client -- predict --target inference
    ```

    The output shows the sample input, the prediction, the device, and the latency that the server reported.

1. Run the following command to benchmark the service with 200 requests, four at a time:

    ```shell
    kubectl exec -n ai-lab deploy/lab-client -- bench --target inference --requests 200 --concurrency 4
    ```

1. Record the results. Exercise 3 compares every profile with these values.

    | Measurement | Your value |
    | :-- | :-- |
    | Latency p50 (ms) | |
    | Latency p95 (ms) | |
    | Throughput (req/s) | |

> [!TIP]
> The benchmark runs inside the cluster, so the results reflect the service rather than your network connection. The p50 value is the median request. The p95 value is the latency that 95 percent of requests beat, which is the tail that users notice. The application reports these values immediately, while dashboards can lag by several minutes.

<details>
<summary>Stuck? Expand this section to catch up with the known-good result of Exercise 2.</summary>

Run the following commands, and then record a baseline with the benchmark command in Task 5:

```shell
kubectl apply -f checkpoints/ex3/
```

```shell
kubectl rollout status deployment/inference -n ai-lab --timeout=180s
```

</details>

## Exercise 3: Optimize AI inference

**Estimated time:** 20 minutes

In this exercise, you size **one pod**. You change the CPU allocated to the inference container and measure the effect. You also start an accelerated profile that runs on a GPU. Replica count comes later, in Exercise 4.

> [!IMPORTANT]
> **Checkpoint:** Before you continue, the **inference** deployment must be running, and your baseline from Exercise 2 must be recorded. If either is missing, expand **Stuck?** at the end of [Exercise 2](#exercise-2-deploy-ai-inference), or ask a proctor.

### Task 1: Start the accelerated profile first

A GPU node doesn't exist until a workload asks for one, and creating one takes several minutes. Start the accelerated profile now so that it's ready when you need it.

1. In VS Code, open **03-optimize/inference-gpu.yaml**, and review the GPU-related settings:

    ```yaml
          nodeSelector:
            karpenter.sh/nodepool: gpu        # only GPU VM sizes with reserved quota
          tolerations:
            - key: nvidia.com/gpu             # GPU nodes repel pods that don't need a GPU
              operator: Exists
              effect: NoSchedule
          containers:
            - name: inference
              image: <registry>/lab723/inference-gpu:1.0.0
              ports:
                - name: http
                  containerPort: 8080
              envFrom:
                - configMapRef:
                    name: lab-environment
                - secretRef:
                    name: appinsights
                    optional: true
              resources:
                requests: { cpu: "2", memory: 8Gi }
                limits:   { cpu: "2", memory: 8Gi, nvidia.com/gpu: 1 }
    ```

1. Run the following command to create the accelerated deployment and its service:

    ```shell
    kubectl apply -f 03-optimize/inference-gpu.yaml
    ```

1. Run the following command to check the new pod:

    ```shell
    kubectl get pods -n ai-lab -l app=inference-gpu
    ```

    The pod is **Pending** because no node in the cluster has a GPU yet.

1. Run the following command, and review the **Events** section at the end of the output:

    ```shell
    kubectl describe pods -n ai-lab -l app=inference-gpu
    ```

    A **FailedScheduling** event from **default-scheduler** reports that no node matches the pod's node selector, because no GPU node exists yet. A later **Nominated** event from **karpenter** names the node that NAP is creating for the pod.

    > **Important:** If the **karpenter** event says **Failed to schedule pod** instead, NAP can't create a GPU node in your environment. Continue with the lab, and tell a proctor.

1. Run the following command to see the node that NAP is creating:

    ```shell
    kubectl get nodeclaims
    ```

    A new NodeClaim shows the GPU VM size in the **TYPE** column. It isn't ready yet. Continue to the next task while it's provisioned.

> [!TIP]
> You didn't choose a VM size, create a node pool, or install GPU drivers. The pod's request for `nvidia.com/gpu: 1` told NAP what to create, the **gpu** `NodePool` limited the choice to a VM size that has quota, and AKS installs the NVIDIA driver and device plugin on the new node. NAP on AKS Automatic is backed by a pod readiness SLA that covers 99.9 percent of qualifying pod readiness operations within five minutes. A GPU workload also pulls a larger image, so allow a few extra minutes.

### Task 2: Measure an undersized profile

1. In VS Code, open **03-optimize/profiles/undersized.yaml**. This patch changes only the CPU allocation and the thread settings:

    ```yaml
    spec:
      template:
        spec:
          containers:
            - name: inference
              env:
                - name: OMP_NUM_THREADS
                  value: "1"
                - name: INFERENCE_THREADS
                  value: "1"
              resources:
                requests: { cpu: 500m, memory: 2Gi }
                limits:   { cpu: 500m, memory: 2Gi }
    ```

1. Run the following command to apply the patch:

    ```shell
    kubectl patch deployment inference -n ai-lab --patch-file 03-optimize/profiles/undersized.yaml
    ```

1. Run the following command, and wait for the new pod to become ready:

    ```shell
    kubectl rollout status deployment/inference -n ai-lab --timeout=180s
    ```

1. Run the benchmark again:

    ```shell
    kubectl exec -n ai-lab deploy/lab-client -- bench --target inference --requests 200 --concurrency 4
    ```

1. Run the following command to read the container's CPU throttling counters:

    ```shell
    kubectl exec -n ai-lab deploy/inference -- cat /sys/fs/cgroup/cpu.stat
    ```

    The `nr_throttled` value counts the scheduling periods in which the container used its entire CPU quota and had to wait. The `throttled_usec` value is the total time that it spent waiting, in microseconds.

1. Record the results:

    | Measurement | Your value |
    | :-- | :-- |
    | Latency p50 (ms) | |
    | Latency p95 (ms) | |
    | Throughput (req/s) | |
    | Throttled periods (`nr_throttled`) | |

> [!TIP]
> With a 500m limit, the container can use half a CPU in every 100-millisecond period. When it uses up that quota, the kernel pauses it until the next period. Requests caught in a pause wait longer, so the tail (p95) grows faster than the median. The pod looks healthy, never restarts, and its average CPU use looks reasonable. The throttling shows up only in latency and in these counters.

### Task 3: Measure a large profile

1. Run the following command to apply the large profile, which allocates two CPUs and two runtime threads:

    ```shell
    kubectl patch deployment inference -n ai-lab --patch-file 03-optimize/profiles/large.yaml
    ```

1. Run the following command, and wait for the new pod to become ready:

    ```shell
    kubectl rollout status deployment/inference -n ai-lab --timeout=180s
    ```

1. Run the benchmark again:

    ```shell
    kubectl exec -n ai-lab deploy/lab-client -- bench --target inference --requests 200 --concurrency 4
    ```

1. Record the results:

    | Measurement | Your value |
    | :-- | :-- |
    | Latency p50 (ms) | |
    | Latency p95 (ms) | |
    | Throughput (req/s) | |

1. (Optional) Run `kubectl exec -n ai-lab deploy/inference -- cat /sys/fs/cgroup/cpu.stat` again, and compare `nr_throttled` with the value for the undersized profile.

### Task 4: Measure the accelerated profile

1. Run the following command to check the accelerated pod and the node it runs on:

    ```shell
    kubectl get pods -n ai-lab -l app=inference-gpu -o wide
    ```

    > **Important:** If the pod isn't **Running** yet, run `kubectl get nodeclaims` to check on the GPU node, continue to Task 5, and return to this task when the pod is running. If the pod is still **Pending** after about eight minutes, skip this task. Your proctor will share GPU results with the room.

1. Run the following command to list the nodes with their VM sizes and node pools:

    ```shell
    kubectl get nodes -L node.kubernetes.io/instance-type,karpenter.sh/nodepool
    ```

    The new node belongs to the **gpu** node pool.

1. Run the following command to confirm that the container can use the GPU:

    ```shell
    kubectl exec -n ai-lab deploy/inference-gpu -- nvidia-smi
    ```

    The output lists the GPU, its memory use, and the inference process that's using it.

1. Run the benchmark against the accelerated service:

    ```shell
    kubectl exec -n ai-lab deploy/lab-client -- bench --target inference-gpu --requests 200 --concurrency 4
    ```

1. Record the results:

    | Measurement | Your value |
    | :-- | :-- |
    | Latency p50 (ms) | |
    | Latency p95 (ms) | |
    | Throughput (req/s) | |

### Task 5: Compare the profiles and choose one

1. Copy the values that you recorded into the following table, and compare them:

    | Profile | CPU limit | Threads | p50 (ms) | p95 (ms) | Throughput (req/s) |
    |:--|--:|--:|--:|--:|--:|
    | Undersized | 500m | 1 | | | |
    | Balanced (your baseline) | 1 | 1 | | | |
    | Large | 2 | 2 | | | |
    | Accelerated | 2 + 1 GPU | n/a | | | |

    <details>
    <summary>What should you notice? Expand this section after you compare your values.</summary>

    - **Undersized:** p95 grows much more than p50. Throttling hurts the tail first.
    - **Balanced to large:** Latency improves, but by much less than the doubling of CPU suggests. Returns diminish once the runtime can't use more parallelism for this model and request size.
    - **Accelerated:** The GPU can deliver the lowest latency and the highest throughput, but it occupies an entire GPU node. It's the right choice only when your latency target or request volume needs it.
    - Your values come from a shared, busy environment. Compare medians and percentiles rather than any single request.

    </details>

1. Discuss how the following factors would change your choice: model size and memory, latency target, expected request volume, availability (several small replicas or one large one), and cost per request.

1. Run the following command to return to the balanced profile, which gets most of the benefit for half the CPU of the large profile:

    ```shell
    kubectl patch deployment inference -n ai-lab --patch-file 03-optimize/profiles/balanced.yaml
    ```

    In Exercise 4, you add capacity horizontally with more replicas of this profile.

1. Open the monitoring dashboard now so that data has time to arrive before Exercise 4. In the search box at the top of the Azure portal, enter `LAB723-AI-inference`, and select the **LAB723-AI-inference** dashboard in the results. This dashboard with Grafana was saved for your lab. At the top, **Prometheus data source** shows your Azure Monitor workspace, **lab723-prometheus**, and **Cluster** shows **lab723-aks**. If either is empty, select it.

    > **Tip:** AKS Automatic also includes Azure-managed dashboards with Grafana for every cluster. To browse them, open your cluster, and under **Monitoring**, select **Dashboards with Grafana**.

> [!TIP]
> **Why the thread settings matter.** Inference runtimes size their thread pools from the number of cores on the node, not from the container's CPU limit. Without the thread settings in each profile, a container limited to one CPU on an eight-core node would start eight threads, be throttled constantly, and might even look slower with more CPU. Matching the thread count to the CPU limit is what makes this comparison fair.

<details>
<summary>Stuck? Expand this section to catch up with the known-good state for Exercise 4.</summary>

```shell
kubectl apply -f checkpoints/ex4/
```

```shell
kubectl rollout status deployment/inference -n ai-lab --timeout=180s
```

</details>

## Exercise 4: Observe and scale

**Estimated time:** 15 minutes

Add a Horizontal Pod Autoscaler (HPA), generate sustained traffic, and watch AKS Automatic add pods and then nodes. Compare what the cluster reports immediately with what the dashboards show.

### Task 1: Inspect the telemetry

1. In the Azure portal, review the **LAB723-AI-inference** dashboard that you opened at the end of Exercise 3. It shows request rate, latency, CPU use compared with limits, replica count, node count, and GPU utilization for the `ai-lab` namespace.

    > **Note:** Metrics take a few minutes to reach the dashboard. If a panel is empty, it's waiting for data, not broken. Use the commands in this exercise for immediate feedback, and return to the dashboard for the overall picture.

    > **Tip:** Managed Prometheus collects the platform's metrics automatically. The application's own metrics, such as request latency, are collected through a `ServiceMonitor` resource in the `azmonitoring.coreos.com/v1` API group. Managed Prometheus ignores the upstream `monitoring.coreos.com/v1` resources that a standard Prometheus Operator manifest creates. The application image is also instrumented with OpenTelemetry, so traces are available without changing any code during the lab.

### Task 2: Add a Horizontal Pod Autoscaler

1. In VS Code, open **04-scale/hpa.yaml**, and review the scaling rules:

    ```yaml
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: inference
      minReplicas: 1
      maxReplicas: 4
      metrics:
        - type: Resource
          resource:
            name: cpu
            target:
              type: Utilization
              averageUtilization: 60       # percent of each pod's CPU request
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 0
        scaleDown:
          stabilizationWindowSeconds: 30   # the default is 300 seconds
          policies:
            - type: Percent
              value: 100
              periodSeconds: 15
    ```

    The shortened scale-down window lets you see the service scale in before the exercise ends.

1. Run the following command to create the autoscaler:

    ```shell
    kubectl apply -f 04-scale/hpa.yaml
    ```

1. Run the following command to check the autoscaler:

    ```shell
    kubectl get hpa inference -n ai-lab
    ```

    The **TARGETS** column might show `<unknown>` for up to a minute while CPU metrics arrive.

### Task 3: Generate sustained traffic

1. In the VS Code terminal panel, select **+** to open a second terminal. In the second terminal, run the following command to watch the autoscaler:

    ```shell
    kubectl get hpa inference -n ai-lab --watch
    ```

1. Switch back to the first terminal, and run the following command to start two load generator pods:

    ```shell
    kubectl scale deployment loadgen -n ai-lab --replicas 2
    ```

### Task 4: Watch pods and nodes scale out

1. In the second terminal, watch the CPU utilization in the **TARGETS** column rise above 60 percent and the **REPLICAS** value increase. This usually takes one to two minutes.

1. In the first terminal, run the following command a few times to see where the new replicas are scheduled:

    ```shell
    kubectl get pods -n ai-lab -l app=inference -o wide
    ```

    The replicas spread across nodes. A replica that has no room stays **Pending** until a new node is ready.

1. Run the following command to see the nodes that NAP creates for the pending replicas:

    ```shell
    kubectl get nodeclaims
    ```

1. Run the following command to compare each replica's CPU use with its one-CPU limit:

    ```shell
    kubectl top pods -n ai-lab -l app=inference
    ```

1. When the replica count stops growing, run `kubectl get nodes`, and record what you observed:

    | Measurement | Your value |
    | :-- | :-- |
    | Highest replica count | |
    | Number of nodes at the peak | |

> [!TIP]
> The topology spread rule in the inference manifest prefers to put each replica on a different node. That improves availability, because losing one node doesn't take down the service. With NAP, it can also create more nodes than packing replicas tightly onto existing nodes would, and every node costs money. Spreading versus bin packing is a trade-off between availability and cost that you control in the manifest.

### Task 5: Compare immediate signals with the dashboard

1. Refresh the **LAB723-AI-inference** dashboard in the Azure portal. Find the increase in request rate, CPU use, and replica count that you just watched in the terminal.

1. Compare the timing. The terminal showed each change within seconds. The dashboard shows the same story a few minutes later, with history and context that the terminal can't give you.

### Task 6: Stop the load and watch the service scale in

1. In the first terminal, run the following command to stop the load generator:

    ```shell
    kubectl scale deployment loadgen -n ai-lab --replicas 0
    ```

1. In the second terminal, watch the CPU utilization fall. After the 30-second stabilization window, **REPLICAS** returns to **1** within a minute or two. Then press **Ctrl+C** to stop watching.

    > **Note:** NAP also removes nodes that are empty or underused, but only after its consolidation checks. You don't need to wait for the extra nodes to disappear.

### Task 7: Release idle GPU capacity

1. On the dashboard, find the GPU utilization panel. With no traffic, the GPU is idle, but the GPU node is still running and still costs money.

1. Run the following command to scale the accelerated deployment to zero:

    ```shell
    kubectl scale deployment inference-gpu -n ai-lab --replicas 0
    ```

1. Run the following command. Within a few minutes, NAP removes the empty GPU node and its NodeClaim.

    ```shell
    kubectl get nodeclaims
    ```

> [!TIP]
> A GPU node costs the same whether it's busy or idle. In production, consider scaling GPU workloads to zero when there's no demand, for example with KEDA, which AKS Automatic enables by default. Limit each `NodePool` to the VM sizes and totals that you have quota for, and reserve capacity for critical events. Quota lets you deploy, but it doesn't guarantee that the hardware is available in the region.

### Task 8: Relate the signals to operational decisions

1. Discuss the following questions with your neighbor or a proctor, and then expand each one to compare answers.

    <details>
    <summary>Which signal would you alert on: CPU utilization, p95 latency, or replica count?</summary>

    Alert on what users experience, such as p95 latency and error rate. Use CPU utilization and replica count to explain why latency changed and to plan capacity.

    </details>

    <details>
    <summary>What limits how far this service can scale?</summary>

    The HPA's maximum replica count, the `NodePool` limits, regional vCPU quota, and the capacity that's actually available in the region. Plan for the peak. When many workloads in a region scale at the same time, quota and capacity become the bottleneck.

    </details>

    <details>
    <summary>How would you lower the cost of this service?</summary>

    Right-size each pod, as you did in Exercise 3. Let the HPA remove idle replicas, keep spreading rules only where availability needs them, and scale accelerators to zero when demand stops.

    </details>

<details>
<summary>Stuck? Expand this section to stop all load and reset the inference service before Exercise 5.</summary>

```shell
kubectl apply -f checkpoints/ex5/
```

</details>

## Exercise 5: Run a distributed AI task

**Estimated time:** 10 minutes

So far, you've run an online service: it's always on, and it answers each request as quickly as possible. Now you run a batch job, which processes a fixed set of inputs as a whole. You use a prepared Jupyter notebook that's connected to the Ray cluster running in the `ray` namespace. You don't write or edit any Python.

> [!TIP]
> **How Ray works.** A Ray cluster has one **head**, which coordinates the work, and one or more **workers**, which do it. The notebook sends a batch to the cluster, Ray splits the batch into tasks, and the workers run the tasks with an existing model. Nothing is trained. On AKS, the KubeRay operator runs the head and each worker as a Kubernetes pod.

### Task 1: Find the Ray cluster's pods

1. Run the following command to list the Ray pods and their roles:

    ```shell
    kubectl get pods -n ray -L ray.io/node-type -o wide
    ```

    The **NODE-TYPE** column shows one **head** pod and several **worker** pods. The **NODE** column shows that they run on the same AKS cluster as the rest of the lab.

### Task 2: Open the notebook

1. In the second terminal, run the following command to connect to the notebook server. Leave the command running.

    ```shell
    kubectl port-forward -n ray svc/lab-notebook 8888:8888
    ```

1. In Edge, open a new tab, and go to `http://localhost:8888/lab/tree/ray-batch-inference.ipynb`.

    > **Note:** The notebook and the Ray cluster are reachable only through this authenticated `kubectl` connection. They aren't exposed to the internet.

### Task 3: Run the prepared cells

1. Read the explanation above each code cell, and then select the cell and press **Shift+Enter** to run it. Run the cells in order:

    1. **Connect:** Connects to the Ray cluster and shows its head and workers.
    1. **Prepare:** Loads a small sample batch and the compact model.
    1. **Run on one worker:** Processes the batch on a single worker.
    1. **Run across all workers:** Splits the same batch across every worker and shows a progress bar.
    1. **Compare:** Shows sample results and a chart of both elapsed times.

1. While the **Run across all workers** cell runs, switch to the first terminal, and run `kubectl top pods -n ray` to see the worker pods using CPU.

1. Record the elapsed times that the notebook reports:

    | Measurement | Your value |
    | :-- | :-- |
    | One worker (seconds) | |
    | All workers (seconds) | |

    > **Note:** For a batch this small, the distributed run might not be faster. Splitting the work, scheduling tasks, and getting the data and model to each worker all take time. Distribution pays off when each item takes longer to process or the batch is much larger.

1. (Optional) To see the work from Ray's point of view, open a third terminal, and run `kubectl port-forward -n ray svc/ray-lab-head-svc 8265:8265`. Then go to `http://localhost:8265` in Edge, and review the **Jobs** and **Cluster** views.

<details>
<summary>Expand this section if the notebook doesn't load or a cell fails.</summary>

Ask a proctor for help. To keep going, open **ray-batch-inference-EXAMPLE-OUTPUT.ipynb** in the same notebook folder. It's a saved copy of a completed run, clearly labeled as an example, so you can read through the results.

</details>

### Task 4: Compare batch and online patterns

1. Consider when each pattern fits, and then expand each question to compare answers.

    <details>
    <summary>How is this batch job different from the inference service?</summary>

    The service is always on, is measured by per-request latency, and scales with the request rate through the HPA. The batch job has a fixed amount of work, is measured by total completion time and throughput, and scales with the size of the data. It can use spare capacity and release it when it finishes.

    </details>

    <details>
    <summary>When does Ray earn its complexity?</summary>

    When a job is too large or too slow for one machine: large batch inference, multistep data pipelines, distributed training or tuning, or pipelines that mix CPU and GPU stages. For work that fits on one node, or for simple fan-out, a Kubernetes Deployment or Job is often enough.

    </details>

1. In each terminal that's running `kubectl port-forward`, press **Ctrl+C** to stop it.

## Exercise 6: Review and next steps

**Estimated time:** 5 minutes

### Task 1: Review what you built

| Decision | What you did | Signals you used |
|:--|:--|:--|
| Make the workload compliant | Fixed a denied image tag, and saw defaults injected into a manifest that omitted them | Admission error message, mutated pod spec |
| Size one pod | Compared undersized, balanced, and large profiles with matching thread settings | p50, p95, throughput, `nr_throttled` |
| Add an accelerator | Requested a GPU, and NAP created a GPU node | Pod events, NodeClaim, `nvidia-smi`, latency |
| Scale out and in | Added an HPA and generated load, and NAP added nodes | `kubectl get hpa`, NodeClaims, dashboard |
| Control cost | Scaled in, and released the idle GPU node | Replica count, GPU utilization, node count |
| Choose a pattern | Ran a batch job with Ray on the same cluster | Elapsed-time comparison |

1. Review the table, and identify one decision that you'd make differently for a workload that you run today.

### Task 2: Take it further

- **Other compute profiles:** To target different hardware, change the `NodePool` requirements and the workload's resource requests. The rest of the manifest stays the same.
- **Other Kubernetes environments:** Explicit requests and limits, probes, versioned image tags, and spreading rules make these manifests portable. On a local cluster, you don't get node autoprovisioning, deployment safeguards, or managed monitoring, so you supply the defaults that AKS Automatic otherwise provides.
- **Takeaway repository:** The manifests, profiles, and notebook from this lab are available at [aka.ms/ignite26-ai-lab](https://aka.ms/ignite26-ai-lab). To run the exercises again in your own subscription, see [Run the exercises on your own](../docs/lab-environment.md#run-the-exercises-on-your-own).

### Task 3: Clean up

1. Make sure that no `--watch` or `kubectl port-forward` command is still running. Press **Ctrl+C** in any terminal that's still busy.

    > **Note:** You don't need to delete anything in this lab. Your environment, including every node that NAP created, is removed when the lab ends.

    <details>
    <summary>Rebuilding this lab in your own subscription? Expand this section for cleanup steps.</summary>

    Delete the resource group that contains your cluster. Deleting the cluster also deletes its node resource group and every node that NAP created:

    ```shell
    az group delete --name <your-resource-group> --yes --no-wait
    ```

    </details>

### Learn more

- [Introduction to AKS Automatic](https://learn.microsoft.com/azure/aks/intro-aks-automatic)
- [Deployment safeguards](https://learn.microsoft.com/azure/aks/deployment-safeguards)
- [Node autoprovisioning overview](https://learn.microsoft.com/azure/aks/node-auto-provisioning) and [node pool configuration](https://learn.microsoft.com/azure/aks/node-auto-provisioning-node-pools)
- [AKS-managed GPU nodes](https://learn.microsoft.com/azure/aks/aks-managed-gpu-nodes)
- [Workload identity on AKS](https://learn.microsoft.com/azure/aks/workload-identity-overview)
- [Monitor AKS](https://learn.microsoft.com/azure/aks/monitor-aks) and [dashboards with Grafana for AKS](https://learn.microsoft.com/azure/azure-monitor/visualize/grafana-kubernetes)
- [Custom Prometheus scrape jobs with CRDs](https://learn.microsoft.com/azure/azure-monitor/containers/prometheus-metrics-scrape-crd)
- [Ray AI workloads on AKS](https://learn.microsoft.com/azure/aks/ray-overview) and [KubeRay](https://docs.ray.io/en/latest/cluster/kubernetes/getting-started.html)

Congratulations, you've completed the lab! If you're in the hosted lab, end the lab from the lab interface when you're finished.
