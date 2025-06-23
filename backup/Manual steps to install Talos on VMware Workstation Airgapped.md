We will be installing a 1-node Kubernetes cluster using Talos Linux. 
This implementaton will be using cilium v1.17.5 instead of the default flannel cni.

## 1. Retrieve container references for system extensions
```bash
$ docker pull ghcr.io/siderolabs/extensions:v1.10.4
$ docker save ghcr.io/siderolabs/extensions:v1.10.4 | cat | grep -A vmtoolsd-guest-agent
ghcr.io/siderolabs/vmtoolsd-guest-agent:v1.0.0@sha256:b80dc11ea312cfe4bf3d8351fe210a76b61c6db04ed30f7b1700d13f473c9e4c
```
## 2. Create installer image to include system extensions and push to image registry
```bash
$ docker run --rm -t -v $PWD/_out:/out ghcr.io/siderolabs/imager:v1.10.4 installer --system-extension-image ghcr.io/siderolabs/vmtoolsd-guest-agent:v1.0.0@sha256:b80dc11ea312cfe4bf3d8351fe210a76b61c6db04ed30f7b1700d13f473c9e4c
$ docker load -i _out/installer-amd64.tar
$ docker tag ghcr.io/siderolabs/installer-base:v1.10.4 sachua/talos-vmware-installer:v1.10.4
$ docker push sachua/talos-vmware-installer:v1.10.4
```
## 3. Create image-cache
```bash
$ talosctl images default > images.txt
$ cat extra-images.txt >> images.txt
$ cat images.txt | talosctl images cache-create --image-cache-path ./image-cache.oci --images=-
```
```yaml
#extra-images.txt
ghcr.io/siderolabs/vmtoolsd-guest-agent:v1.0.0@sha256:b80dc11ea312cfe4bf3d8351fe210a76b61c6db04ed30f7b1700d13f473c9e4c
sachua/talos-vmware-installer:v1.10.4
quay.io/cilium/cilium-cli:v0.18.4
quay.io/cilium/cilium:v1.17.5
quay.io/cilium/certgen:v0.2.1
quay.io/cilium/hubble-relay:v1.17.5
quay.io/cilium/hubble-ui-backend:v0.13.2
quay.io/cilium/hubble-ui:v0.13.2
quay.io/cilium/cilium-envoy:v1.32.6-1749271279-0864395884b263913eac200ee2048fd985f8e626
quay.io/cilium/operator:v1.17.5
quay.io/cilium/startup-script:c54c7edeab7fde4da68e59acd319ab24af242c3f
quay.io/cilium/clustermesh-apiserver:v1.17.5
docker.io/library/busybox:1.37.0
ghcr.io/spiffe/spire-agent:1.9.6
ghcr.io/spiffe/spire-server:1.9.6
```
## 4. Create iso with image-cache
```bash
docker run --rm -t -v $PWD/_out:/secureboot:ro -v $PWD/_out:/out -v $PWD/image-cache.oci:/image-cache.oci:ro -v /dev:/dev --privileged ghcr.io/siderolabs/imager:v1.10.4 iso --image-cache /image-cache.oci
```
## 5. Create VM on VMware Workstation using the metal-amb64.iso
Minimum Requirements
| Cores | Memory | Storage |
| :---: | :---: | :---: |
| 2 | 2 GiB | 10 GB |
## 6. Create and bootstrap Kubernetes on Talos
```
talosctl gen secrets -o secret.yaml
talosctl gen config --with-secrets secrets.yaml --config-patch-control-plane @cp1.patch --output-types controlplane -o cp1.yaml my-cluster https://${REPLACE_WITH_ENDPOINT_IP}:6443
talosctl apply-config --insecure --nodes ${REPLACE_WITH_NODE_IP} --file cp1.yaml
```

```yaml
machine:
  install:
    image: sachua/talos-vmware-installer:v1.10.4
  features:
    hostDNS:
      forwardKubeDNSToHost: true
    imageCache:
      localEnabled: true
  network:
    hostname: control-plane-1
    interfaces:
    - interface: ens33
      dhcp: true
      vip:
        ip: ${REPLACE_WITH_ENDPOINT_IP}
cluster:
  apiServer:
    extraArgs:
      feature-gates: AuthorizeNodeWithSelectors=false # Caused by K8s 1.32.0 enabling AuthorizeNodeWithSelectors feature gate
  network:
    cni:
      name: none
  proxy:
    disabled: true
  discovery:
    enabled: true
    registries:
      kubernetes:
        disabled: false
      service:
        disabled: true
  allowSchedulingOnControlPlanes: true
  inlineManifests:
    - name: cilium-install
      contents: |
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: cilium-install
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: cluster-admin
        subjects:
        - kind: ServiceAccount
          name: cilium-install
          namespace: kube-system
        ---
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: cilium-install
          namespace: kube-system
        ---
        apiVersion: batch/v1
        kind: Job
        metadata:
          name: cilium-install
          namespace: kube-system
        spec:
          backoffLimit: 10
          template:
            metadata:
              labels:
                app: cilium-install
            spec:
              restartPolicy: OnFailure
              tolerations:
                - operator: Exists
                - effect: NoSchedule
                  operator: Exists
                - effect: NoExecute
                  operator: Exists
                - effect: PreferNoSchedule
                  operator: Exists
                - key: node-role.kubernetes.io/control-plane
                  operator: Exists
                  effect: NoSchedule
                - key: node-role.kubernetes.io/control-plane
                  operator: Exists
                  effect: NoExecute
                - key: node-role.kubernetes.io/control-plane
                  operator: Exists
                  effect: PreferNoSchedule
              affinity:
                nodeAffinity:
                  requiredDuringSchedulingIgnoredDuringExecution:
                    nodeSelectorTerms:
                      - matchExpressions:
                          - key: node-role.kubernetes.io/control-plane
                            operator: Exists
              serviceAccount: cilium-install
              serviceAccountName: cilium-install
              hostNetwork: true
              containers:
              - name: cilium-install
                image: quay.io/cilium/cilium-cli:v0.18.4
                env:
                - name: KUBERNETES_SERVICE_HOST
                  valueFrom:
                    fieldRef:
                      apiVersion: v1
                      fieldPath: status.podIP
                - name: KUBERNETES_SERVICE_PORT
                  value: "6443"
                command:
                  - cilium
                  - install
                  - --version
                  - 1.17.5
                  - --set
                  - ipam.mode=kubernetes
                  - --set
                  - kubeProxyReplacement=true
                  - --set
                  - securityContext.capabilities.ciliumAgent={CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}
                  - --set
                  - securityContext.capabilities.cleanCiliumState={NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}
                  - --set
                  - cgroup.autoMount.enabled=false
                  - --set
                  - cgroup.hostRoot=/sys/fs/cgroup
                  - --set
                  - k8sServiceHost=localhost
                  - --set
                  - k8sServicePort=7445
---
apiVersion: v1alpha1
kind: VolumeConfig
name: IMAGECACHE
provisioning:
  diskSelector:
    match: 'system_disk'
  minSize: 2GB
  maxSize: 2GB
```