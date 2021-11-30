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
