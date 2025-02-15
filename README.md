# Jenkins Deployment with Kubernetes

## Pre-requisites
### 1. Create a role
- Go to AWS console and open the IAM Console.
- On the left side, select **Role**
- In the search bar, enter the name of your cluster
- Select the role with nodegroup mentioned inside the role name
- Click on Add permissions then Create Inline Policy
- On the Policy Editor Pane; select JSON and paste the follow

```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateVolume",
                "ec2:AttachVolume",
                "ec2:DetachVolume",
                "ec2:DeleteVolume",
                "ec2:DescribeVolumes",
                "ec2:DescribeVolumeStatus",
                "ec2:ModifyVolume",
                "ec2:CreateTags"  
            ],
            "Resource": "*"
        }
    ]
}
```
- Click on the Next button
- Enter the Name and save

### 2. Delete the default Service Class
The default Service Class here is delete to create another one on EBS.
- Delete the exixting default Service Class

```bash
kubectl delete sc gp2
```

## Deployment of Jenkins
**1. Create a namespace**

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: devops-tools
```

```bash
kubectl apply -f 1.namespace.yaml
```

**2. Create a service account**

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
  namespace: devops-tools
```

```bash
kubectl apply -f 2.serviceaccount.yaml
```

**3. Create a Cluster Role**

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-admin
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["*"]
```

```bash
kubectl apply -f 3.clusterrole.yaml
```

**4. Create a Cluster Role Binding**

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-admin
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: devops-tools
```

```bash
kubectl apply -f 4.clusterrolebinding.yaml
```

**5. Install drivers and utilities for EBS volume**

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=master
```

**6. Create Volume**

```bash
# Storage Class definition for EBS
# Volume will be created in the Cloud Provider, here is AWS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage # This is just a name, but the StorageClass is created in the Coud Provider
provisioner: ebs.csi.aws.com  # Ensure the AWS EBS CSI driver is installed
parameters:
  type: gp2
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pv-claim
  namespace: devops-tools
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
```

```bash
kubectl apply -f 5.volume.yaml
```

**7. Create Deployment**

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins-server
  template:
    metadata:
      labels:
        app: jenkins-server
    spec:
      securityContext:
            fsGroup: 1000 
            runAsUser: 1000
      serviceAccountName: jenkins-admin
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          resources:
            limits:
              memory: "2Gi"
              cpu: "1000m"
            requests:
              memory: "500Mi"
              cpu: "500m"
          ports:
            - name: httpport
              containerPort: 8080
            - name: jnlpport
              containerPort: 50000
          livenessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 90
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: "/login"
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: jenkins-data
              mountPath: /var/jenkins_home         
      volumes:
        - name: jenkins-data
          persistentVolumeClaim:
              claimName: jenkins-pv-claim
```

```bash
kubectl apply -f 6.deployment.yaml
```

**7. Create Service**

```bash
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: devops-tools
spec:
  selector:
    app: jenkins-server
  ports:
    - name: httpport
      port: 80
      targetPort: 8080
    - name: jnlpport
      port: 50000
      targetPort: 50000
  type: LoadBalancer
```

```bash
kubectl apply -f 7.service.yaml
```