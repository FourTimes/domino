# domino

General requirements

      1. Kubernetes 1.13+
      2. Cluster permissions
      3. Three namespaces
          3.1 Domino creates three dedicated namespaces
            3.1.1 one for Platform nodes
            3.1.2 one for Compute nodes
            3.1.3 one for installer metadata and secrets
      4. Storage classes
          4.1 Domino requires at least two storage classes.
            4.1.1 Dynamic block storage
            4.1.2 Long term shared storage
           
Node pool requirements

  Domino requires a minimum of two node pools
  
    1. one to host the Domino Platform
    2. one to host Compute workloads.
    3. Additional optional pools can be added to provide specialized execution hardware for some Compute workloads.

##### Platform pool requirements

    Boot Disk: 128GB
    Min Nodes: 3 (4 if Domino Model Monitor is enabled)
    Max Nodes: 3 (4 if Domino Model Monitor is enabled)
    Spec: 8 CPU / 32GB
    Labels: 
        dominodatalab.com/node-pool: platform
    Tags:
        kubernetes.io/cluster/{{ cluster_name }}: owned
        k8s.io/cluster-autoscaler/enabled: true #Optional for autodiscovery
        k8s.io/cluster-autoscaler/{{ cluster_name }}: owned #Optional for autodiscovery
        
##### Compute pool requirements

    Boot Disk: 400GB
    Recommended Min Nodes: 1
    Max Nodes: Set as necessary to meet demand and resourcing needs
    Recommended min spec: 8 CPU / 32GB
    Enable Autoscaling: Yes
    Labels: 
        domino/build-node: true
        dominodatalab.com/node-pool: default
    Tags:
        k8s.io/cluster-autoscaler/node-template/label/dominodatalab.com/node-pool: default
        kubernetes.io/cluster/{{ cluster_name }}: owned
        k8s.io/cluster-autoscaler/node-template/label/domino/build-node: true
        k8s.io/cluster-autoscaler/enabled: true #Optional for autodiscovery
        k8s.io/cluster-autoscaler/{{ cluster_name }}: owned #Optional for autodiscovery

##### Optional GPU compute pool

      Boot Disk: 400GB
      Recommended Min Nodes: 0
      Max Nodes: Set as necessary to meet demand and resourcing needs
      Recommended min Spec: 8 CPU / 16GB / One or more Nvidia GPU Device
      Nodes must be pre-configured with appropriate Nvidia driver, Nvidia-docker2 and set the default docker runtime to nvidia. For example, EKS GPU optimized AMI.
      Labels: 
            dominodatalab.com/node-pool: default-gpu, 
            nvidia.com/gpu: true
      Tags:
            k8s.io/cluster-autoscaler/node-template/label/dominodatalab.com/node-pool: default-gpu
            kubernetes.io/cluster/{{ cluster_name }}: owned
            k8s.io/cluster-autoscaler/enabled: true #Optional for autodiscovery
            k8s.io/cluster-autoscaler/{{ cluster_name }}: owned #Optional for autodiscovery

##### cluster verification process

      You should perform the following steps from a workstation with kubectl admin access to the target cluster.

##### Install Sonobuoy binaries

      If the cluster is running Kubernetes 1.13, install sonobuoy v0.15.4.
      If the cluster is running Kubernetes 1.14, install sonobuoy v0.16.2
      If the cluster is running Kubernetes 1.15 or above, install the latest sonobuoy release

```bash
 wget https://github.com/vmware-tanzu/sonobuoy/releases/download/v0.55.1/sonobuoy_0.55.1_linux_amd64.tar.gz
 tar -xvzf sonobuoy_0.55.1_linux_amd64.tar.gz
 sudo mv sonobuoy /usr/local/bin/
```


Run the following command to determine the Kubernetes version for your cluster:

      kubectl version

Set a KUBECONFIG environment variable to a path to a kubeconfig file with admin access to the target cluster.

      export KUBECONFIG=~/.kube/config
      
Create a `domino-checker.yaml` configuration file with the following contents

```yml
# domino-checker.yaml
sonobuoy-config:
 driver: DaemonSet
 plugin-name: domino
 result-format: junit
 skip-cleanup: true
spec:
 env:
 - name: DOCKER_API_VERSION
   value: '1.38'
 - name: NODE_NAME
   valueFrom:
     fieldRef:
       fieldPath: spec.nodeName
 - name: POD_NAME
   valueFrom:
     fieldRef:
       fieldPath: metadata.name
 - name: POD_NAMESPACE
   valueFrom:
     fieldRef:
       fieldPath: metadata.namespace
 - name: RESULTS_DIR
   value: /tmp/results
 image: quay.io/domino/k8s-validator:latest
 imagePullPolicy: Always
 name: domino
 securityContext:
   privileged: false
 volumeMounts:
 - mountPath: /tmp/results
   name: results
   readOnly: false
 - mountPath: /var/run/docker.sock
   name: docker-mnt
   readOnly: false

extra-volumes:
- name: docker-mnt
  hostPath:
   path: /var/run/docker.sock
```


Commands

```bash
      sonobuoy run -p domino-checker.yaml --wait
      resultsfile=$(sonobuoy retrieve)
      sonobuoy results $resultsfile --plugin domino
      sonobuoy delete --wait
```

```yaml
---
# cluster.yml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: domino
  region: us-east-2
nodeGroups:
  - name: domino-platform
    instanceType: m5.2xlarge
    minSize: 3
    maxSize: 3
    desiredCapacity: 3
    volumeSize: 128
    availabilityZones: ["us-east-2a"]
    labels:
      "dominodatalab.com/node-pool": "platform"
    tags:
      "k8s.io/cluster-autoscaler/enabled": "true" 
      "k8s.io/cluster-autoscaler/domino": "owned" 
  - name: domino-default
    instanceType: m5.2xlarge
    minSize: 0
    maxSize: 10
    desiredCapacity: 1
    volumeSize: 400
    availabilityZones: ["us-east-2a"]
    labels:
      "dominodatalab.com/node-pool": "default"
      "domino/build-node": "true"
    tags:
      "k8s.io/cluster-autoscaler/node-template/label/dominodatalab.com/node-pool": "default"
      "k8s.io/cluster-autoscaler/node-template/label/domino/build-node": "true"
      "k8s.io/cluster-autoscaler/enabled": "true" 
      "k8s.io/cluster-autoscaler/domino": "owned" 
    preBootstrapCommands:
      - "cp /etc/docker/daemon.json /etc/docker/daemon_backup.json"
      - "echo -e '.bridge=\"docker0\" | .\"live-restore\"=false' >  /etc/docker/jq_script"
      - "jq -f /etc/docker/jq_script /etc/docker/daemon_backup.json | tee /etc/docker/daemon.json"
      - "systemctl restart docker"
  - name: domino-gpu
    instanceType: p2.8xlarge
    minSize: 0
    maxSize: 5
    volumeSize: 400
    availabilityZones: ["us-east-2a"]
    ami:
      ami-014b0ee6a978b6ca5
    labels:
      "dominodatalab.com/node-pool": "default-gpu"
      "nvidia.com/gpu": "true"
    tags:
      "k8s.io/cluster-autoscaler/node-template/label/dominodatalab.com/node-pool": "default-gpu"
      "k8s.io/cluster-autoscaler/enabled": "true" 
      "k8s.io/cluster-autoscaler/domino": "owned" 
availabilityZones: ["us-east-2a", "us-east-2b", "us-east-2c"]
```
output

`1`

![image](https://user-images.githubusercontent.com/57703276/144083172-d8880879-3950-49c8-89f4-477ccbd1f429.png)

`2`

![image](https://user-images.githubusercontent.com/57703276/144083009-39629b58-2c66-4d6d-bcb9-250f253db4f2.png)
