# K8s-Study

# Kubernetes Revision Guide for Developers

## Table of Contents
- [Core Architecture](#core-architecture)
- [Workload Resources](#workload-resources)
- [Scheduling & Placement](#scheduling--placement)
- [Networking](#networking)
- [Storage](#storage)
- [Extensions & Custom Resources](#extensions--custom-resources)
- [Developer Tools & Troubleshooting](#developer-tools--troubleshooting)

## Core Architecture
- **Control Plane** (Master Components)
  - `api-server`: API entry point for all operations
  - `etcd`: Distributed key-value store for cluster state
  - `scheduler`: Assigns pods to nodes
  - `controller-manager`: Maintains desired state
  - `cloud-controller-manager`: Interfaces with cloud providers

- **Node Components**
  - `kubelet`: Node agent ensuring containers run in pods
  - `kube-proxy`: Network proxy, handles service networking
  - `container-runtime`: (Docker/containerd) Runs containers

## Workload Resources

### Pods
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: nginx:latest
    ports:
    - containerPort: 80
```
- Smallest deployable unit
- Can contain multiple containers
- Share network namespace and storage

### Deployments
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      securityContext:           # Pod-level security settings
        runAsNonRoot: true      # Ensures no container runs as root
        fsGroup: 2000           # Sets filesystem group for all containers
      containers:
      - name: myapp
        image: nginx:latest
        securityContext:         # Container-level security settings
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsUser: 1000       # Run as UID 1000
          runAsGroup: 3000      # Run as GID 3000
          capabilities:
            drop: ["ALL"]       # Drop all capabilities
            add: ["NET_BIND_SERVICE"]  # Only add required capabilities
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        volumeMounts:           # Required when using readOnlyRootFilesystem
        - name: tmp-volume      # For temporary files
          mountPath: /tmp
        - name: var-run
          mountPath: /var/run
      volumes:                  # Define writable volumes
      - name: tmp-volume
        emptyDir: {}
      - name: var-run
        emptyDir: {}
```
- Manages ReplicaSets
- Handles rolling updates
- Maintains desired state

**Resource Properties Explained:**
- `requests`: Minimum resources guaranteed to the container
  - Used by scheduler to find suitable nodes
  - Important for proper pod placement
  - Should reflect actual application needs

- `limits`: Maximum resources the container can use
  - Container will be throttled if exceeding CPU limit
  - Container will be OOM killed if exceeding memory limit
  - Should be set to prevent resource hogging

- CPU Units:
  - Expressed in cores or millicores
  - "1" = 1 CPU core
  - "500m" = 500 millicores = 0.5 CPU core
  - "250m" = 250 millicores = 0.25 CPU core

- Memory Units:
  - Expressed in bytes: Ki, Mi, Gi
  - "64Mi" = 64 Mebibytes
  - "1Gi" = 1 Gibibyte
  - Can also use K, M, G (decimal) units

**Security Context Properties Explained:**
- Pod-level security (applies to all containers):
  - `runAsNonRoot`: Prevents containers from running as root
  - `fsGroup`: Sets the group ID for mounted volumes

- Container-level security:
  - `readOnlyRootFilesystem`: Makes root filesystem read-only
  - `runAsUser`: Specifies the UID to run the container
  - `runAsGroup`: Specifies the GID to run the container
  - `allowPrivilegeEscalation`: Controls if process can gain more privileges
  - `capabilities`: Fine-grained Linux capabilities control
    - `drop: ["ALL"]`: Removes all capabilities for security
    - `add: []`: Only add specific required capabilities

**Note:** When using `readOnlyRootFilesystem: true`, you need to:
1. Mount emptyDir volumes for directories requiring write access (e.g., /tmp)
2. Configure your application to use these writable mounts
3. Ensure all required directories are either read-only or mounted as volumes

### StatefulSets
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mydb
spec:
  serviceName: mydb  # Headless service for network identity
  replicas: 3
  selector:          # Must match template labels, like in Deployments
    matchLabels:
      app: mydb
  template:
    metadata:
      labels:        # Must match selector.matchLabels
        app: mydb
    spec:
      containers:
      - name: mydb
        image: mysql:5.7
        volumeMounts:        # References volumes defined below
        - name: data        # Must match volume name in volumeClaimTemplates
          mountPath: /var/lib/mysql
  volumeClaimTemplates:     # Creates PVC for each pod
  - metadata:
      name: data           # Referenced by volumeMounts.name above
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```
- For stateful applications
- Stable network identities
- Ordered scaling/updates
- Persistent storage per pod

### DaemonSets
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
      - name: agent
        image: monitoring-agent:latest
```
- Runs one pod per node
- Used for node-level operations
- Common for logging/monitoring

## Scheduling & Placement

### Node Selectors
```yaml
spec:
  nodeSelector:
    disk: ssd
```
- Simple node selection
- Based on node labels

### Node Affinity/Anti-Affinity
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: zone
            operator: In
            values:
            - zone1
```
- More expressive than nodeSelector
- Supports complex matching
- Can be required or preferred

### Pod Affinity/Anti-Affinity
```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: kubernetes.io/hostname
```
- Schedule pods relative to other pods
- Based on topology domains

### Taints & Tolerations
```yaml
# Node Taint
kubectl taint nodes node1 key=value:NoSchedule

# Pod Toleration
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```
- Taints prevent pod scheduling on nodes
- Tolerations allow pods on tainted nodes

## Networking

### Services & Pod Connection
Services use label selectors to find and route traffic to pods. The relationship works as follows:

1. Pods are labeled with key-value pairs
2. Services use selectors to match these labels
3. The Service automatically discovers and routes traffic to matching pods

```yaml
# Example Pod with labels
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:            # Labels that can be selected by Services
    app: myapp      
    tier: frontend
spec:
  containers:
  - name: myapp
    image: nginx:latest
    ports:
    - containerPort: 80

---
# Service selecting the pod
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: ClusterIP  # or LoadBalancer/NodePort
  selector:        # Selects pods with matching labels
    app: myapp    # Must match pod labels exactly
    tier: frontend
  ports:
  - port: 80        # Port the service listens on
    targetPort: 80  # Port on the pod to forward to
```

For Deployments/ReplicaSets, the label relationship is:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3
  selector:          # Deployment uses this to find its pods
    matchLabels:     # Must match template labels
      app: myapp
      tier: frontend
  template:
    metadata:
      labels:        # Labels applied to pods created by this deployment
        app: myapp
        tier: frontend
    spec:
      containers:
      - name: myapp
        image: nginx:latest
        ports:
        - containerPort: 80

---
# Service will automatically find all pods created by the deployment
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: ClusterIP
  selector:        # Matches labels in deployment's pod template
    app: myapp
    tier: frontend
  ports:
  - port: 80
    targetPort: 80
```

Key Points about Service-to-Pod Mapping:
1. **Label Selection**:
   - Services use label selectors to dynamically discover pods
   - Any pod with matching labels will automatically be added to the service's endpoint list
   - Multiple pods with matching labels will receive traffic according to the service's load balancing rules

2. **Why Selectors in Deployments**:
   - Deployments use selectors to track which pods they own
   - The selector must match the pod template labels
   - This creates a three-way relationship: Service → Pod Labels ← Deployment Selector

3. **Dynamic Updates**:
   - When pods are added/removed, services automatically update their endpoints
   - No manual configuration needed for service discovery
   - Makes scaling and updates seamless

4. **Endpoint Verification**:
   ```bash
   # Check if service is finding pods
   kubectl get endpoints myapp-service
   
   # Detailed view of service
   kubectl describe service myapp-service
   ```

### Services
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: ClusterIP  # Determines service exposure level
  selector:        # Finds pods to send traffic to
    app: myapp    # Must match pod labels
  ports:
  - port: 80       # Port exposed by the service
    targetPort: 8080  # Port on the pod to forward to
```
- Types: ClusterIP, NodePort, LoadBalancer
- Service discovery and load balancing
- Stable network endpoint

### Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
  - host: myapp.example.com  # External hostname
    http:
      paths:
      - path: /             # URL path to match
        pathType: Prefix
        backend:
          service:
            name: myapp-service  # Must match existing Service name
            port:
              number: 80        # Must match Service port
```
- L7 load balancing
- TLS termination
- Name-based virtual hosting

## Storage

### Volumes & VolumeMounts
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: nginx
    volumeMounts:           # References volumes defined below
    - name: config-volume   # Must match volume name
      mountPath: /etc/config
    - name: data-volume    # Must match volume name
      mountPath: /data
  volumes:                 # Volume definitions
  - name: config-volume    # Referenced by volumeMounts.name
    configMap:
      name: app-config    # Must match existing ConfigMap name
  - name: data-volume     # Referenced by volumeMounts.name
    persistentVolumeClaim:
      claimName: myapp-pvc  # Must match existing PVC name
```
- Types: emptyDir, hostPath, configMap, secret, persistentVolumeClaim
- Lifecycle tied to pod
- Support for multiple volume types

### PersistentVolume (PV)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: myapp-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:  # Example backend, use cloud provider's storage in production
    path: /mnt/data
```
- Cluster-level storage resource
- Independent of pod lifecycle
- Provisioned by admin or dynamically

### PersistentVolumeClaim (PVC)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```
- Request for storage by pod
- Can specify size and access mode
- Binds to available PV

### StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs  # Example for AWS
parameters:
  type: gp2
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```
- Defines storage profiles
- Enables dynamic provisioning
- Provider-specific parameters

### Common Storage Patterns

1. **Dynamic Provisioning**
```yaml
# StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2

# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: fast
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

2. **Shared Storage (ReadWriteMany)**
```yaml
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs  # Example using NFS

# Pod using shared storage
apiVersion: v1
kind: Pod
metadata:
  name: shared-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    persistentVolumeClaim:
      claimName: shared-pvc
```

### Storage Troubleshooting

1. **Common Issues**
   - PVC stuck in Pending
   - Volume mount permission issues
   - Storage capacity issues
   - Volume not detaching

2. **Debug Commands**
```bash
# Check PV/PVC status
kubectl get pv,pvc
kubectl describe pv <pv-name>
kubectl describe pvc <pvc-name>

# Check StorageClass
kubectl get sc
kubectl describe sc <storage-class-name>

# Debug pod volume issues
kubectl describe pod <pod-name>
kubectl get events --field-selector involvedObject.name=<pvc-name>
```

3. **Best Practices**
   - Use dynamic provisioning when possible
   - Set appropriate storage requests
   - Consider volume expansion requirements
   - Use appropriate access modes
   - Implement proper backup strategies

## Extensions & Custom Resources

### Custom Resource Definitions (CRDs)
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.myapp.com
spec:
  group: myapp.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    kind: Backup
```
- Extend Kubernetes API
- Define custom resources

### Operators
- Automate application management
- Built using CRDs and controllers
- Handle application-specific logic

## Developer Tools & Troubleshooting

### Common Commands
```bash
# Logs
kubectl logs pod-name -c container-name
kubectl logs -f pod-name  # Follow logs

# Debug
kubectl describe pod pod-name
kubectl get events --sort-by=.metadata.creationTimestamp

# Exec into container
kubectl exec -it pod-name -- /bin/bash

# Port forwarding
kubectl port-forward pod-name 8080:80
```

### Common Issues & Solutions

1. **Pod Stuck in Pending**
   - Check node resources: `kubectl describe node`
   - Check pod events: `kubectl describe pod`
   - Verify node selectors/affinity rules

2. **Pod Stuck in CrashLoopBackOff**
   - Check logs: `kubectl logs pod-name`
   - Verify resource limits
   - Check liveness/readiness probes

3. **Image Pull Errors**
   - Verify image name/tag
   - Check image pull secrets
   - Ensure registry access

4. **Service Discovery Issues**
   - Verify service selectors match pod labels
   - Check endpoints: `kubectl get endpoints`
   - Test using service DNS: `service-name.namespace.svc.cluster.local`

5. **Network Connectivity**
   - Use network debug pods
   - Check network policies
   - Verify service/pod IP ranges

### Best Practices for Developers

1. **Resource Management**
   - Always set resource requests/limits
   - Use horizontal pod autoscaling
   - Monitor resource usage

2. **Application Configuration**
   - Use ConfigMaps/Secrets
   - Implement proper health checks
   - Handle SIGTERM gracefully

3. **Debugging Tools**
   - `kubectl debug`
   - `ksniff` for packet capture
   - `stern` for multi-pod logs

4. **Local Development**
   - Use Minikube/Kind
   - Implement dev/prod parity
   - Use Skaffold for development workflow
