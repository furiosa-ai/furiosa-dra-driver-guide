# Furiosa DRA Driver Guide
This guide provides instructions for enabling Dynamic Resource Allocation (DRA) in Kubernetes v1.33, installing Furiosa DRA driver and requesting Furiosa NPU through DRA API.

## Installing v1.33 Kubernetes Binaries

Run the following commands to install Kubernetes v1.33 binaries on Ubuntu
See [Installing Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) for more details.
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

## Enable CDI feature for ContainerD 
To use the DRA API, CDI must be enabled in the container runtime.

ContainerD supports CDI by default starting from version 2.0.0.
For earlier versions, you need to explicitly enable CDI in the config.toml configuration file, typically located at /etc/containerd/config.toml.

To enable CDI support, add the following configuration:
```yaml
[plugins."io.containerd.grpc.v1.cri"]
enable_cdi = true
cdi_spec_dirs = ["/etc/cdi", "/var/run/cdi"]
```

After modifying the configuration, restart ContainerD to apply the changes:
```bash
sudo systemctl restart containerd
```

For more details, refer to the official ContainerD documentation on [enabling CDI support](https://github.com/containerd/containerd/blob/main/docs/cri/config.md).

## Kubeadm Configuration for DRA Enablement in Kubernetes v1.33 

DRA is a Beta feature in v1.33, usually a beta feature is enabled by default, but DRA introduces so many API changes so it is not enabled by default in v1.33.
To enable DRA in Kubernetes v1.33, you need to explicitly enable few feature gates for api-server, controller-manager, scheduler and kubelet.
These configuration won't be necessary in v1.34, as DRA will be promoted to GA in v1.34 and enabled by default.

Following instructions and example `kubeadm` configuration illustrate how to enable DRA related feature gates for each component in Kubernetes v1.33.
Please use valid settings for your cluster in all parts of ClusterConfiguration and InitConfiguration, except for the apiServer, controllerManager, and scheduler sections in ClusterConfiguration, and the kubeletExtraArgs setting in InitConfiguration.

```bash
cat <<EOF > config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: "v1.33.2"
networking:
  podSubnet: "10.244.0.0/16"
apiServer:
  extraArgs:
    - name: feature-gates
      value: "DynamicResourceAllocation=true,DRAResourceClaimDeviceStatus=true"
    - name: runtime-config
      value: "resource.k8s.io/v1beta1=true,resource.k8s.io/v1beta2=true"
controllerManager:
  extraArgs:
    - name: feature-gates
      value: "DynamicResourceAllocation=true"
scheduler:
  extraArgs:
    - name: feature-gates
      value: "DynamicResourceAllocation=true"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
nodeRegistration:
  criSocket: "/var/run/containerd/containerd.sock"
  kubeletExtraArgs:
    - name: feature-gates
      value: "DynamicResourceAllocation=true"
skipPhases:
  - addon/kube-proxy
  
sudo kubeadm init --config dra.yaml
```

Once your cluster is successfully initialized you can simply verify DRA enablement with following command
```bash
kubectl get deviceclasses
```

If the result is similar to the following, then DRA is enabled successfully.
```bash
No resources found
```

If you see an error like `error: the server doesn't have a resource type "deviceclasses"`, then DRA is not enabled successfully, you may need to check the kubeadm configuration.

You can also enable the DRA feature manually by first creating the cluster using the default kubeadm settings, and then editing the configuration files of the kube-apiserver, kube-controller-manager, kube-scheduler, and kubelet directly to enable the required feature gates and API groups.
See [enabling dynamic resource allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#enabling-dynamic-resource-allocation) for more details.

## Installing Furiosa DRA Driver
Experimental version of Furiosa DRA Driver and Helm Chart for 2025.2.0 release are available now.
- Dockerhub: https://hub.docker.com/r/furiosaai/furiosa-dra-driver/tags
- HelmChart: https://github.com/furiosa-ai/helm-charts/tree/main/charts/furiosa-dra-driver

Following instructions illustrate how to install Furiosa DRA driver in Kubernetes v1.33 with Helm chart.
We recommend to install Furiosa DRA Driver along with Node Feature Discovery and furiosa-feature-discovery to ensure the DRA driver can discover Furiosa NPU devices correctly.

Following instructions illustrate how to add the Helm repository and install Furiosa DRA driver with Node Feature Discovery and furiosa-feature-discovery.
```bash
helm repo add furiosa https://furiosa-ai.github.io/helm-charts
helm repo update
helm install -n kube-system furiosa-feature-discovery furiosa/furiosa-feature-discovery --version=2025.2.0
helm install -n kube-system furiosa-dra-driver furiosa/furiosa-dra-driver --version=2025.2.0
```

If you successfully installed Furiosa DRA driver, you can verify the installation with following commands.
```bash
kubectl get deviceclasses
NAME             AGE
npu.furiosa.ai   92m

kubectl get resourceslice
NAME                                 NODE            DRIVER           POOL            AGE
rngd-tsvr-010-npu.furiosa.ai-bdb5l   rngd-tsvr-010   npu.furiosa.ai   rngd-tsvr-010   92m
```

If you want to deploy only Furiosa DRA driver without Node Feature Discovery and Furiosa Feature Discovery deployment, you can skip installing them.
However, you need to remove the affinity section from the [values.yaml](https://github.com/furiosa-ai/helm-charts/blob/main/charts/furiosa-dra-driver/values.yaml) file of Furiosa DRA driver Helm chart.
```yaml
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: feature.node.kubernetes.io/pci-1200_1ed2.present
                operator: In
                values:
                  - "true"
```

### Requesting Furiosa NPU via DRA API

DRA API is inspired by the Persistent Volume API. So even though you are new to DRA API, you can easily understand its API pattern and usage.
We are going to use following new Kubernetes API to request Furiosa NPU devices.
- DeviceClass: Defines a class of devices, such as "npu.furiosa.ai".
- ResourceSlice: Defines a collection of devices and its attributes and capacity.
- ResourceClaim: Defines a claim for a specific resource, such as a Furiosa NPU device.
- ResourceClaimTemplate: Defines a template for creating ResourceClaims, it is optional but recommended for some use cases such as requesting multiple devices with the same configuration.

Visit [Dynamic Resource Allocation - API](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#api) for more details about DRA API.

#### Requesting Furiosa NPU using ResourceClaimTemplate
The following example shows how to request a Furiosa NPU using a ResourceClaimTemplate.
This approach is convenient when requesting an NPU without specific requirements.

In this example, we define a basic ResourceClaimTemplate that can be reused by multiple Pods.
Each time a Pod references the ResourceClaimTemplate to request a device, a new ResourceClaim is automatically created and bound to that Pod.
When the Pod is deleted, the associated ResourceClaim is automatically cleaned up as well.

Run the following command to create a ResourceClaimTemplate and two Pods that request a Furiosa NPU device.
```bash
kubectl apply -f - <<EOF
apiVersion: resource.k8s.io/v1beta1
kind: ResourceClaimTemplate
metadata:
  name: single-npu
spec:
  spec:
    devices:
      requests:
        - name: npu
          deviceClassName: npu.furiosa.ai
---
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    app: pod1
spec:
  containers:
    - name: furiosa
      image: furiosaai/furiosa-smi:2025.2.0
      command:
        - sleep
        - "86400"
      resources:
        claims:
          - name: npu
  resourceClaims:
    - name: npu
      resourceClaimTemplateName: single-npu
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  labels:
    app: pod2
spec:
  containers:
    - name: furiosa
      image: furiosaai/furiosa-smi:2025.2.0
      command:
        - sleep
        - "86400"
      resources:
        claims:
          - name: npu
  resourceClaims:
    - name: npu
      resourceClaimTemplateName: single-npu
EOF
```

If you apply the above YAML, it will create a ResourceClaimTemplate named `single-npu`, and two Pods named `pod1` and `pod2` that request a Furiosa NPU device using this template.
You can also see the automatically created ResourceClaims for each Pod.
```bash
kubectl get resourceclaimtemplate
NAME         AGE
single-npu   12s

kubectl get resourceclaim
NAME             STATE                AGE
pod1-npu-hdbmx   allocated,reserved   17s
pod2-npu-jlfw9   allocated,reserved   17s

kubectl get pod
NAME   READY   STATUS    RESTARTS   AGE
pod1   1/1     Running   0          26s
pod2   1/1     Running   0          26s
```

You can also check the details of the resource allocation.
In this result, `status.allocation.devices[0].results[0].device` indicates that the RNGD(`npu3`) of `rngd-tsvr-010` node is allocated for the ResourceClaim `pod1-npu-hdbmx` associated with `pod1`.
```bash
kubectl get resourceclaim pod1-npu-hdbmx -o yaml
apiVersion: resource.k8s.io/v1beta2
kind: ResourceClaim
metadata:
  annotations:
    resource.kubernetes.io/pod-claim-name: npu
  creationTimestamp: "2025-07-03T00:13:48Z"
  finalizers:
  - resource.kubernetes.io/delete-protection
  generateName: pod1-npu-
  name: pod1-npu-hdbmx
  namespace: default
  ownerReferences:
  - apiVersion: v1
    blockOwnerDeletion: true
    controller: true
    kind: Pod
    name: pod1
    uid: 7cae21e9-1d91-47ef-81fe-655d3b8ca9b9
  resourceVersion: "22604"
  uid: dbe93324-0afc-4cc2-83a1-ff9387ac38f8
spec:
  devices:
    requests:
    - exactly:
        allocationMode: ExactCount
        count: 1
        deviceClassName: npu.furiosa.ai
      name: npu
status:
  allocation:
    devices:
      results:
      - adminAccess: null
        # this field indicates which device is allocated for this ResourceClaim
        device: npu3
        driver: npu.furiosa.ai
        pool: rngd-tsvr-010
        request: npu
    nodeSelector:
      nodeSelectorTerms:
      - matchFields:
        - key: metadata.name
          operator: In
          values:
          - rngd-tsvr-010
  reservedFor:
  - name: pod1
    resource: pods
    uid: 7cae21e9-1d91-47ef-81fe-655d3b8ca9b9
```

#### Requesting Furiosa NPU using ResourceClaim with specific requirements
In some cases, you may want to request a Furiosa NPU with specific requirements, such as:
- a specific device identified by UUID or Name
- devices located in a specific NUMA node
- devices using a specific driver version or firmware version

In this example, we request specific Furiosa NPU devices by UUID.
Instead of using a ResourceClaimTemplate, we define each ResourceClaim explicitly, linking it to a unique device.

For demonstration, we create two ResourceClaims, each associated with a specific device:
- npu0: UUID 40512C86-0705-4506-8E45-494644454A43
- npu1: UUID 41512C86-0703-4208-8C43-49444C4C4E43

Kubernetes DRA supports using CEL expressions in the selectors field of a ResourceClaim to match devices based on their attributes.
For example, to select a specific device by UUID, you can use the following expression:
```yaml
expression: "device.attributes['npu.furiosa.ai'].uuid == '09512C86-070D-4204-8342-4C424747474B'"
```

Run the following command to create two ResourceClaims and two Pods, where each Pod is configured to requests a specific Furiosa NPU device through its associated ResourceClaim.
```bash
kubectl apply -f - <<EOF
apiVersion: resource.k8s.io/v1beta1
kind: ResourceClaim
metadata:
  name: npu0-claim
spec:
  devices:
    requests:
      - name: rngd-npu
        deviceClassName: npu.furiosa.ai
        allocationMode: ExactCount
        selectors:
          - cel:
              expression: "device.attributes['npu.furiosa.ai'].uuid == '40512C86-0705-4506-8E45-494644454A43'"
---
apiVersion: resource.k8s.io/v1beta1
kind: ResourceClaim
metadata:
  name: npu1-claim
spec:
  devices:
    requests:
      - name: rngd-npu
        deviceClassName: npu.furiosa.ai
        allocationMode: ExactCount
        selectors:
          - cel:
              expression: "device.attributes['npu.furiosa.ai'].uuid == '41512C86-0703-4208-8C43-49444C4C4E43'"
---
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    app: pod1
spec:
  containers:
    - name: furiosa
      image: furiosaai/furiosa-smi:2025.2.0
      command:
        - sleep
        - "86400"
      resources:
        claims:
          - name: npu
  resourceClaims:
    - name: npu
      resourceClaimName: npu0-claim
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  labels:
    app: pod2
spec:
  containers:
    - name: furiosa
      image: furiosaai/furiosa-smi:2025.2.0
      command:
        - sleep
        - "86400"
      resources:
        claims:
          - name: npu
  resourceClaims:
    - name: npu
      resourceClaimName: npu1-claim
EOF
```

Following command will show the created ResourceClaims and Pods.
You can also check the `status.allocation.devices[0].results[0].device` field of each ResourceClaim to see which device is allocated to each Pod.
```bash
kubectl get pod
NAME   READY   STATUS    RESTARTS   AGE
pod1   1/1     Running   0          2m12s
pod2   1/1     Running   0          2m12s

kubectl get resourceclaim
NAME         STATE                AGE
npu0-claim   allocated,reserved   11s
npu1-claim   allocated,reserved   11s
```

Note that when a ResourceClaim is created explicitly, it is not automatically deleted even if the Pod referencing it is removed.
In the example below, each Pod that references a ResourceClaim is deleted. The Claims are then released, but remain in the Pending state:
```bash
kubectl delete pod pod1 pod2
pod "pod1" deleted
pod "pod2" deleted

kubectl get resourceclaim
NAME         STATE     AGE
npu0-claim   pending   5m26s
npu1-claim   pending   5m26s
```

This approach makes it possible to pre-create ResourceClaim objects for specific devices, such as those identified by UUID or name, and have Pods consume them only when needed.
It follows a usage pattern similar to how PersistentVolumeClaims (PVCs) are used in Kubernetes: you can declare the resource ahead of time and bind it to a Pod later, giving you more control and flexibility over device allocation.
