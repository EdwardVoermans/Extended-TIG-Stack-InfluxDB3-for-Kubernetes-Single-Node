# Custom Storage Class Configuration for K3s

## Overview

This guide explains how to configure a custom storage class in K3s to control where Persistent Volume Claims (PVCs) store their data on your local filesystem.

## The Problem

By default, K3s uses the `local-path` storage class, which creates PVC data in an auto-generated path like:
```
/var/lib/kubelet/pods/d508db95-a968-48b3-b85a-706dd520cd7c/volumes/kubernetes.io~local-volume
```

The GUID is generated automatically, making it difficult to manage and locate your persistent data.

### Historical Context

In older K3s versions (pre-1.26), you could use a simple annotation to override the storage path:

```yaml
annotations:
  local-path-provisioner.k3s.cattle.io/path: "/data/k3s-pvc"
```

**This annotation method is no longer supported in K3s v1.26+**, so we need to create a custom storage class instead.

## Solution: Custom Storage Class

### Step 1: Create the Data Directory

First, create your preferred storage location:

```bash
sudo mkdir -p /data/k3s-pvc
sudo chmod 755 /data/k3s-pvc
```

### Step 2: Create the ConfigMap and StorageClass

Create a file named `local-path-config.yaml`:

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-path-config
  namespace: kube-system
data:
  config.json: |
    {
      "nodePathMap":[
        {
          "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
          "paths":["/data/k3s-pvc"]
        }
      ]
    }

  setup: |-
    #!/bin/sh
    set -eu
    mkdir -m 0777 -p "${VOL_DIR}"
    chmod 755 "${VOL_DIR}/.."

  teardown: |-
    #!/bin/sh
    set -eu
    rm -rf "${VOL_DIR}"

  helperPod.yaml: |-
    apiVersion: v1
    kind: Pod
    metadata:
      name: helper-pod
    spec:
      containers:
      - name: helper-pod
        image: "rancher/mirrored-library-busybox:1.36.1"
        imagePullPolicy: IfNotPresent
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: custom-local-path
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

Apply the configuration:

```bash
kubectl apply -f local-path-config.yaml
```

### Step 3: Restart the Local Path Provisioner

**This step is critical and required:**

```bash
kubectl -n kube-system delete pod -l app=local-path-provisioner
```

You should see output like:
```
pod "local-path-provisioner-774c6665dc-xp8tp" deleted
```

Kubernetes will automatically create a new local-path-provisioner pod with your new configuration.

## Verification

### Create a Test PVC

Create `test-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-local-path
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: custom-local-path
  resources:
    requests:
      storage: 1Gi
```

### Create a Test Pod

Create `test-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: default
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - mountPath: /mnt/data
          name: test-volume
  volumes:
    - name: test-volume
      persistentVolumeClaim:
        claimName: test-local-path
```

Apply both files:

```bash
kubectl apply -f test-pvc.yaml
kubectl apply -f test-pod.yaml
```

### Verify the Setup

Check that the PVC is bound:

```bash
kubectl get pvc
```

Expected output:
```
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
test-local-path   Bound    pvc-24634628-2c05-44e4-9225-4269845e77ff   1Gi        RWO            custom-local-path   27m
```

Verify the data is stored in your custom location:

```bash
ls -l /data/k3s-pvc/
```

Expected output:
```
total 4
drwxrwxrwx 2 root root 4096 Nov 18 08:17 pvc-24634628-2c05-44e4-9225-4269845e77ff_default_test-local-path
```

### Cleanup

Remove the test resources:

```bash
kubectl delete pod test-pod
kubectl delete pvc test-local-path
```

## Usage in Your Applications

From now on, use the following in your PVC specifications:

```yaml
spec:
  storageClassName: custom-local-path
```

This will ensure all your persistent volumes are created in your preferred `/data/k3s-pvc` directory instead of the default Kubernetes location.

## Key Points

- The custom storage class name is `custom-local-path`
- Always restart the local-path-provisioner after applying the configuration
- The `volumeBindingMode: WaitForFirstConsumer` ensures the PV is only created when a pod uses it
- The `reclaimPolicy: Delete` means data is deleted when the PVC is removed
- You can customize the storage path by changing `/data/k3s-pvc` to any location you prefer
