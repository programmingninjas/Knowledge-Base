
## Overview

Kamaji turns any Kubernetes cluster into a **Management Cluster**, orchestrating other Kubernetes clusters known as **Tenant Clusters**. It runs Control Plane components inside pods instead of dedicated machines, making it cost-effective and easier to operate.
![[Kamaji Architecture.png]]
- For detailed concepts, refer to the [Kamaji documentation](https://kamaji.clastix.io/concepts/).

---

## **Prerequisites**

1. **Kubernetes Cluster**: A pre-initialized cluster with configurations supporting CAPI.
2. **CSI driver:** I used Longhorn as the CSI driver. Refer to the [Longhorn Documentation](https://longhorn.io/docs/1.7.2/deploy/install/install-with-helm/).
3. **Cert-Manager**: A TLS management tool for securing webhooks and communication.
4. For Kubeadm I used this config:

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    cloud-provider: "external"
  name: <Master-Node-Name>
localAPIEndpoint:
  advertiseAddress: <Private-IP>
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
apiServer:
  certSANs:
    - "<Public-IP>"
    - "<Private-IP>"
  extraArgs:
    cloud-provider: "external"
networking:
  podSubnet: "10.244.0.0/16" # Default for Flannel
controllerManager:
  extraArgs:
    cloud-provider: "external"
```
## **Installation Steps**

### 1. **Install Cert-Manager**

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version v1.11.0 \
    --set installCRDs=true
```

### 2. **Install Kamaji via Helm**

```bash
helm repo add clastix https://clastix.github.io/charts
helm repo update
helm install kamaji clastix/kamaji -n kamaji-system --create-namespace
```

### 3. **Verify Installation**

```bash
kubectl -n kamaji-system get pods
```

### 4. **Install Kamaji Console**

- Ensure you configure the required secret before adding console:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kamaji-console
  namespace: kamaji-system
stringData:
  ADMIN_EMAIL: <email>
  ADMIN_PASSWORD: <password>
  JWT_SECRET: <jwtSecret>
  NEXTAUTH_URL: <nextAuthUrl>
```

- Add the console via Helm:

```bash
helm -n kamaji-system install console clastix/kamaji-console --values values.yml
```

- Using values.yml to expose console service with ingress:

```yaml
ingress:
  enabled: true
  className: "nginx" # Specify the ingress class; change as per your ingress controller.
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod" # Example for TLS with Cert-Manager
    cert-manager.io/acme-challenge-type: "http01"
    cert-manager.io/acme-challenge-solver-class: "<class>"
  hosts:
    - host: kamaji.example.com # Replace with your actual hostname
      paths:
        - path: /ui
          pathType: ImplementationSpecific
  tls:
    - secretName: kamaji-console-tls # Secret for TLS certificates
      hosts:
        - kamaji.example.com # Must match the above hostname

```

---

## **OpenStack Cloud Controller Manager**

### 1. **Create Cloud Configuration Secret**

```bash
#cloud.conf  
[Global] 
auth-url=  
username=  
password= 
region= 
tenant-id= 
tenant-domain-id= 
domain-id=
[LoadBalancer]
subnet-id= #Of your private network
floating-network-id=
```

```bash
kubectl create secret -n kube-system generic cloud-config --from-file=cloud.conf
```

### 2. **Apply Resources**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/cloud-controller-manager-roles.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/cloud-controller-manager-role-bindings.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/openstack-cloud-controller-manager-ds.yaml
```

---

## **Clusterctl: Initialize Management Cluster**

1. **Install Clusterctl**

```bash
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.8.5/clusterctl-linux-amd64 -o clusterctl
sudo install -o root -g root -m 0755 clusterctl /usr/local/bin/clusterctl
clusterctl version
```

2. **Initialize Management Cluster**

```bash
clusterctl init --infrastructure openstack --control-plane kamaji --bootstrap kubeadm
```

---
### **Create Workload Cluster**

#### 1. **Download `clouds.yaml` File**

- Obtain the `clouds.yaml` file from the **OpenStack Horizon Dashboard** under the **API Access** section.
- Convert the file into base64 encoding:

```bash
base64 clouds.yaml > clouds.b64
```

#### 2. **Create a Secret for Cloud Config**

- Use the base64-encoded file to create a Kubernetes secret:

```bash
kubectl create secret -n kube-system generic cloud-config --from-file=cloud.conf
```

#### 3. **Workload Cluster Manifest**

- Use the CAPI-compatible image required for bootstrapping.

- Example manifest for a workload cluster:

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: capi-quickstart
  namespace: default
spec:
  clusterNetwork:
    serviceDomain: cluster.local
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1alpha1
    kind: KamajiControlPlane
    name: kamaji-quickstart-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: OpenStackCluster
    name: capi-quickstart
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: OpenStackCluster
metadata:
  name: capi-quickstart
  namespace: default
spec:
  apiServerLoadBalancer:
    enabled: false
  controlPlaneAvailabilityZones:
    - <Zone>
  disableAPIServerFloatingIP: true
  disableExternalNetwork: true
  apiServerFixedIP: ""
  network:
    id: <External-Network-Id> or <Preferred-Network-Id>
  subnets:
    - id: <External-Subnet-Id> or <Preferred-Subnet-Id>
  identityRef:
    name: capi-quickstart
    cloudName: openstack
  managedSecurityGroups:
    allowAllInClusterTraffic: false
    allNodesSecurityGroupRules:
      - remoteManagedGroups:
          - worker
        direction: ingress
        etherType: IPv4
        name: BGP
        portRangeMin: 179
        portRangeMax: 179
        protocol: "tcp"
        description: "Allow BGP among workers"
      - remoteManagedGroups:
          - worker
        direction: ingress
        etherType: IPv4
        name: IP-in-IP
        protocol: "4"
        description: "Allow IP-in-IP among workers"
---
apiVersion: controlplane.cluster.x-k8s.io/v1alpha1
kind: KamajiControlPlane
metadata:
  name: kamaji-quickstart-control-plane
  namespace: default
spec:
  replicas: <Number-of-Control-Planes>
  version: <Kubernetes-Version>
  dataStoreName: default
  addons:
    coreDNS: {}
    kubeProxy: {}
    konnectivity: {}
  kubelet:
    preferredAddressTypes:
      - InternalIP
      - ExternalIP
      - Hostname
  network:
    serviceType: LoadBalancer
  controllerManager:
    extraArgs:
      - --cloud-provider=external
  apiServer:
    extraArgs:
      - --cloud-provider=external
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: capi-quickstart-md-0
  namespace: default
spec:
  clusterName: capi-quickstart
  replicas: <Number-of-Workers>
  template:
    spec:
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: capi-quickstart-md-0
      clusterName: capi-quickstart
      failureDomain: <Zone>
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: OpenStackMachineTemplate
        name: capi-quickstart-md-0
      version: <Kubernetes-Version>
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: OpenStackMachineTemplate
metadata:
  name: capi-quickstart-md-0
  namespace: default
spec:
  template:
    spec:
      flavor: <Flavor>
      rootVolume:
        sizeGiB: <Size>
      availabilityZone:
        name: <Zone>
      image:
        id: <Image-ID>
      sshKeyName: <KeyPair-Name>
---
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata:
  name: capi-quickstart-md-0
  namespace: default
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration:
          kubeletExtraArgs:
            cloud-provider: external
          name: '{{ local_hostname }}'
---
apiVersion: v1
kind: Secret
metadata:
  name: capi-quickstart
  namespace: default
data:
  clouds.yaml: <Base64-clouds.yaml>
```

- Save the manifest as `capi-quickstart.yaml` and apply it:

```bash
kubectl apply -f capi-quickstart.yaml
```

#### 4. **Post-Creation Steps**

- Fetch the kubeconfig for the workload cluster:

```bash
kubectl get secret <kamaji-control-plane>-kubeconfig -o jsonpath='{.data.value}' | base64 --decode > workload-kubeconfig.yml
```

- Install the CNI (e.g., Flannel):

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml --kubeconfig workload-kubeconfig.yml
```

- Install the OpenStack Cloud Controller Manager (OCCM):

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/cloud-controller-manager-roles.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/cloud-controller-manager-role-bindings.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/openstack-cloud-controller-manager-ds.yaml
```

#### 5. **Modify the OCCM DaemonSet**

- To schedule the OCCM pods on worker nodes (since control plane nodes are managed by Kamaji), edit the daemonset to remove the following:

```yaml
nodeSelector:
  node-role.kubernetes.io/control-plane: ""
tolerations:
  - effect: NoSchedule
    key: node.cluster.x-k8s.io/uninitialized
```

---
## References

- [Kamaji Concepts](https://kamaji.clastix.io/concepts/)
- [OpenStack Cloud Provider Documentation]([cloud-provider-openstack/docs/openstack-cloud-controller-manager/using-openstack-cloud-controller-manager.md at master · kubernetes/cloud-provider-openstack](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/openstack-cloud-controller-manager/using-openstack-cloud-controller-manager.md))
- [Cluster API OpenStack Images](https://github.com/osism/k8s-capi-images)
