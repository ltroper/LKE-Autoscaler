# LKE-Autoscaler

## Using Linode Kubernetes Engine (LKE) Autoscaler

This guide demonstrates how to set up and test node autoscaling on Linode Kubernetes Engine (LKE). Node autoscaling automatically adjusts the number of nodes in your cluster based on resource demands, helping optimize costs and performance.

### Prerequisites
- Linode account
- `kubectl` installed on your local machine
- Basic understanding of Kubernetes concepts

---

### Step 1: Deploy LKE Cluster
1. Log into your Linode Cloud Manager
2. Navigate to Kubernetes in the left menu
3. Click "Create Cluster"
4. Choose your preferred region
5. Select node pool specifications (CPU, RAM, etc.)


### Step 2: Enable Autoscaler
Under "Node Pool", enable "Autoscale Pool"
   - Set minimum nodes (e.g., 2)
   - Set maximum nodes (e.g., 5)
   - This allows LKE to automatically add or remove nodes based on resource utilization

### Step 3: Connect to Your Cluster
1. Download your kubeconfig file from the Linode Cloud Manager
2. Move the file to your local machine:
   ```bash
   mv ~/Downloads/kubeconfig.yaml ~/.kube/config
   export KUBECONFIG=~/.kube/config
   ```
3. Verify connection:
   ```bash
   kubectl get nodes
   ```

### Step 4: Deploy Test Workload
We'll deploy a CPU-intensive workload to trigger the autoscaler:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-spinner
spec:
  selector:
    matchLabels:
      app: cpu-spinner
  replicas: 1
  template:
    metadata:
      labels:
        app: cpu-spinner
    spec:
      containers:
      - name: cpu-spinner
        image: busybox
        resources:
          requests:
            cpu: "200m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "128Mi"
        command: ["/bin/sh"]
        args:
        - "-c"
        - "while true; do dd if=/dev/zero of=/dev/null bs=1M count=1024; done"
```

Save this as `cpu-spinner.yaml` and apply:
```bash
kubectl apply -f cpu-spinner.yaml
```

#### Understanding the Test Deployment:
- Uses `busybox` image for a lightweight container
- Runs a continuous CPU-intensive operation using `dd` command
- Each pod requests 200mCPU (20% of a CPU core) and is limited to 500mCPU
- The `dd` command continuously writes data from `/dev/zero` to `/dev/null`, creating CPU load
- Memory requests are set to 64Mi with a limit of 128Mi

### Step 5: Test Scaling Up
1. Scale the deployment to 20 replicas:
   ```bash
   kubectl scale deployment cpu-spinner --replicas=20
   ```
2. Monitor node creation:
   ```bash
   kubectl get nodes
   # or watch in real-time:
   watch kubectl get nodes
   ```

What's happening:
- The increased number of pods creates resource pressure
- When existing nodes can't accommodate new pods due to CPU/memory requests
- LKE autoscaler detects this and provisions new nodes

### Step 6: Test Scaling Down
1. Reduce the deployment to 2 replicas:
   ```bash
   kubectl scale deployment cpu-spinner --replicas=2
   ```
2. Monitor node removal:
   ```bash
   watch kubectl get nodes
   ```

What's happening:
- Reduced pod count means less resource demand
- Autoscaler identifies underutilized nodes
- After a cooldown period (typically 10-15 minutes)
- Excess nodes are safely drained and removed

### Clean Up
Remove the test deployment:
```bash
kubectl delete deployment cpu-spinner
```

---


This setup provides automatic scaling of your Kubernetes infrastructure based on actual demand, optimizing resource usage and costs.
