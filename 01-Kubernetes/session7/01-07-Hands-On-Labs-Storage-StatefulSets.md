# Session 01-07: Storage - Volumes, PV/PVC & StatefulSets - Hands-On Labs

---

## Pre-requisite: AWS EBS CSI Driver Installation

> **Note:** The EBS CSI driver can also be installed via **AWS EKS Managed Add-Ons** from the AWS Console. The steps below cover the manual installation method using Helm.

### Step 1: Create an IAM Role

- Open the IAM console at https://console.aws.amazon.com/iam/.
- In the left navigation pane, choose **Roles**.
- On the Roles page, choose **Create role**.
- On the Select trusted entity page, do the following:
  - In the Trusted entity type section, choose **Web identity**.
  - For Identity provider, choose the **OpenID Connect provider URL** for your cluster (as shown under Overview in Amazon EKS).
  - For Audience, choose **sts.amazonaws.com**.
  - Choose **Next**.
- On the Add permissions page, do the following:
  - In the Filter policies box, enter **AmazonEBSCSIDriverPolicy**.
  - Select the check box to the left of the **AmazonEBSCSIDriverPolicy** returned in the search.
  - Choose **Next**.
- On the Name, review, and create page, do the following:
  - For Role name, enter a unique name for your role, such as **AmazonEKS_EBS_CSI_DriverRole**.
  - Choose **Create role**.
- After the role is created, choose the role in the console to open it for editing.
- Choose the **Trust relationships** tab, and then choose **Edit trust policy**.
- Find the line that looks similar to the following line:

```
"oidc.eks.region-code.amazonaws.com/id/CF856D2CC9C5E229C4C6D3D43B178C5E:aud": "sts.amazonaws.com"
```

- Add a comma to the end of the previous line, and then add the following line after it. Replace `region-code` with the AWS Region that your cluster is in. Replace `CF856D2CC9C5E229C4C6D3D43B178C5E` with your cluster's OIDC provider ID.

```
"oidc.eks.region-code.amazonaws.com/id/CF856D2CC9C5E229C4C6D3D43B178C5E:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
```

- Choose **Update policy** to finish.

### Step 2: Install the EBS CSI Driver via Helm

```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
helm upgrade --install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver -n kube-system
```

### Step 3: Verify the Driver Pods are Running

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

Once the driver pods are in `Running` state, you are ready to proceed with the storage labs below.

---

## Lab 1: StorageClass & Dynamic Provisioning

**Objective:** Use StorageClass to automatically create PersistentVolumes when PVCs are requested.

### Steps:

1. Check available StorageClasses:

```bash
kubectl get storageclass
```

2. Create a StorageClass (for EBS in AWS):

Create manifest file storage-class.yaml

```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
allowVolumeExpansion: true
```

```bash
kubectl apply -f storage-class.yaml
kubectl get storageclass
```

3. Create a PVC that references the StorageClass:


Create manifest file pvc-dynamic.yaml

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3-sc
  resources:
    requests:
      storage: 5Gi
```

```bash
kubectl apply -f pvc-dynamic.yaml
```


**Access Modes:**
   - RWO (ReadWriteOnce): One pod reads & writes (EBS)
   - ROX (ReadOnlyMany): Many pods read only (NFS/EFS)
   - RWX (ReadWriteMany): Many pods read & write (NFS/EFS)


4. Watch the PV be auto-created:

```bash
kubectl get pvc dynamic-pvc -w
```
# Status changes from Pending → Bound in ~10-30 seconds

```bash
kubectl get pv
```
# A new PV should appear, automatically created!

```bash
kubectl describe pvc dynamic-pvc
```

5. Create a deployment using the PVC:

create manifest file deployment-dynamic-pvc.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-dynamic-storage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-with-dynamic-storage
  template:
    metadata:
      labels:
        app: app-with-dynamic-storage
    spec:
      containers:
      - name: app
        image: busybox
        volumeMounts:
        - name: data
          mountPath: /data
        command: ["sh", "-c", "echo 'Dynamic storage' >> /data/note.txt; sleep 1000"]
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: dynamic-pvc
```

```bash
kubectl apply -f deployment-dynamic-pvc.yaml
kubectl exec -it app-with-dynamic-storage -- cat /data/note.txt
```

**Key Takeaway:** StorageClass enables self-service provisioning without admin intervention. Keep replicas: 1 unless your PVC supports multi-pod access, for example ReadWriteMany.

6. Now let us explore the benefits of PVC

-> The Pod is temporary, but the storage is persistent. Since our Deployment is using the same PVC, if we delete the Pod, Kubernetes will create a new Pod, attach the same volume again, and the data inside /data will still be available.

```bash
kubectl get pods
```

Add some extra data manually:
```bash
kubectl exec -it <pod-name-from-above> -- sh
```

Inside the pod:
```bash
echo "This data should survive pod deletion" >> /data/note.txt
cat /data/note.txt
exit
```

Now delete the Pod:
```bash
kubectl delete pod <same-pod-name-from-above>
```

The Deployment will create a new Pod automatically:
```bash
kubectl get pods
kubectl exec -it <pod-name-from-here> -- cat /data/note.txt
```

You should see the same data!!

---

## Lab 5: Deploy a StatefulSet - MySQL with Persistent Storage

**Objective:** Deploy a MySQL StatefulSet to demonstrate stable pod identity, stable DNS names, ordered startup, and unique persistent storage per replica.

### Steps:

Follow the details mentioned in the kubernetes.io page
https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/

This lab should focus on following the kubernetes.io documentation

---

## Lab 6: Test StatefulSet Behavior

**Objective:** Verify pod identity persistence, ordering, and DNS resolution.

### Steps:

1. **Test 1: Pod Identity Persistence**

```bash
# Get the current pod
kubectl get pod mysql-0 -o wide

# Note the node it's running on

# Delete mysql-0
kubectl delete pod mysql-0

# Watch it recreate with the same name
kubectl get pods -l app=mysql -w

# mysql-0 should reappear on the same (or different) node
# But it will bind to the SAME PVC (mysql-data-mysql-0)
```

2. **Test 2: Ordered Scaling**

```bash
# Scale down
kubectl scale statefulset mysql --replicas=2
kubectl get pods -l app=mysql
# mysql-0, mysql-1 remain; mysql-2 is deleted

# Scale up
kubectl scale statefulset mysql --replicas=3
# mysql-2 is recreated (in order)

kubectl get pods -l app=mysql
```

3. **Test 3: Headless Service DNS**

```bash
# Launch a test pod
kubectl run -it --rm dns-test --image=busybox --restart=Never -- sh

# Inside the container:
nslookup mysql-0.mysql-headless.default.svc.cluster.local
nslookup mysql-1.mysql-headless.default.svc.cluster.local
nslookup mysql-2.mysql-headless.default.svc.cluster.local

# Compare with regular service DNS (notice no pod name):
nslookup mysql-headless.default.svc.cluster.local
# This returns multiple A records (one per pod)
```

---

## Cleanup

```bash
kubectl delete statefulset mysql
kubectl delete svc mysql-headless
kubectl delete configmap mysql-config
kubectl delete storageclass mysql-storage
kubectl delete pvc --all
kubectl delete pod --all

# Optional: Clean up manually created resources
kubectl delete pv,pvc --all
```

---

## Key Concepts to Remember

1. **emptyDir:** Ephemeral, shared within a pod, deleted on pod termination
2. **PersistentVolume (PV):** Cluster-level storage resource
3. **PersistentVolumeClaim (PVC):** Pod's request for storage (1-to-1 binding with PV)
4. **StorageClass:** Automates PV creation via CSI drivers
5. **Access Modes:**
   - RWO (ReadWriteOnce): One pod reads & writes (EBS)
   - ROX (ReadOnlyMany): Many pods read only (NFS)
   - RWX (ReadWriteMany): Many pods read & write (NFS)
6. **StatefulSet:** Provides stable pod identity, ordered scaling, unique PVC per pod
7. **Headless Service:** Direct DNS to individual pods (clusterIP: None)
8. **volumeClaimTemplates:** Automatically create unique PVCs for each StatefulSet replica

---

## Additional Labs to cover by self:

## Lab 5: emptyDir Volume - Container-to-Container Sharing

**Objective:** Understand how emptyDir allows containers in the same pod to share ephemeral data.

### Steps:

1. Create a pod with emptyDir and two containers:

```bash
cat > pod-emptydir-lab.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: writer
    image: busybox
    volumeMounts:
    - name: shared-data
      mountPath: /data
    command: ["sh", "-c", "for i in {1..5}; do echo 'Line '$i >> /data/shared.txt; sleep 2; done; sleep 100"]
  - name: reader
    image: busybox
    volumeMounts:
    - name: shared-data
      mountPath: /reader
    command: ["sh", "-c", "sleep 5; while true; do echo '--- Reader output ---'; cat /reader/shared.txt 2>/dev/null || echo 'File not yet written'; sleep 10; done"]
  volumes:
  - name: shared-data
    emptyDir: {}
EOF

kubectl apply -f pod-emptydir-lab.yaml
```

2. Watch both containers:

```bash
# Terminal 1: Watch writer
kubectl logs -f emptydir-demo -c writer

# Terminal 2: Watch reader
kubectl logs -f emptydir-demo -c reader
```

3. Verify they're sharing data (reader sees writer's output):

```bash
kubectl exec -it emptydir-demo -c reader -- cat /reader/shared.txt
```

4. Delete the pod and observe data is lost:

```bash
kubectl delete pod emptydir-demo
# Data is gone forever - emptyDir is ephemeral
```

**Key Takeaway:** emptyDir is perfect for inter-container communication within a pod but is destroyed when the pod terminates.

---

## Lab 6: ConfigMap as Volume - Mount Configuration Files

**Objective:** Mount a ConfigMap as a volume to serve static files from nginx.

### Steps:

1. Create a ConfigMap with HTML content:

```bash
cat > html-content.html <<EOF
<!DOCTYPE html>
<html>
<head><title>Storage Demo</title></head>
<body>
<h1>Hello from ConfigMap Volume!</h1>
<p>This HTML is stored in a ConfigMap and mounted as a volume.</p>
<p>Time: $(date)</p>
</body>
</html>
EOF

kubectl create configmap html-content --from-file=html-content.html
kubectl describe configmap html-content
```

2. Create a pod that mounts the ConfigMap:

```bash
cat > pod-configmap-vol-lab.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configmap-demo
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    volumeMounts:
    - name: html-vol
      mountPath: /usr/share/nginx/html
  volumes:
  - name: html-vol
    configMap:
      name: html-content
EOF

kubectl apply -f pod-configmap-vol-lab.yaml
```

3. Port-forward and test:

```bash
kubectl port-forward pod/nginx-configmap-demo 8080:80 &
curl http://localhost:8080/html-content.html
kill %1
```

4. Update the ConfigMap and verify the pod sees the change (within 1 minute):

```bash
kubectl edit configmap html-content
# Change the <p> text to something else

# Wait ~1 minute and refresh
kubectl port-forward pod/nginx-configmap-demo 8080:80 &
curl http://localhost:8080/html-content.html
```

**Key Takeaway:** ConfigMap volumes are read-only and updates propagate with a delay. Good for config files but not for data that pods write to.

---