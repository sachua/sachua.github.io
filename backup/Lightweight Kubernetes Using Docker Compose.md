Local development on Kubernetes can be a hassle since the resources required to run a full Kubernetes cluster can be quite heavy.

In this post, we will explore running a single node Kubernetes cluster for your local development needs using Docker Compose.

## Kubernetes Distribution

We will be focusing on 2 different Kubernetes distributions: `K3s` and `RKE2`.

[K3s](https://k3s.io/) is a Rancher's lightweight, fully compliant Kubernetes distribution, that is packaged into a single binary. It is most commonly used for edge computing or IoT use cases, and there are projects such as [k3d](https://k3d.io/) which is a lightweight wrapper to run K3s in docker.

[RKE2](https://rke2.io/) is Rancher's next-generation Kubernetes distribution, combining RKE1's close alignment with upstream Kubernetes and K3s's deployment model for ease-of-operations. It also comes with FIPS 140-2 compliance.

## K3s in Docker

k3d provides a simple solution to create k3s clusters for local development, but we are looking for an even simpler solution that only uses Docker and Docker Compose.

You only need a `docker-compose.yml` file:

```YAML
version: '3.7'
services:
  server:
    image: rancher/k3s:v1.24.0-rc1-k3s1-amd64
    networks:
    - default
    command: server
    tmpfs:
    - /run
    - /var/run
    ulimits:
      nproc: 65535
      nofile:
        soft: 65535
        hard: 65535
    privileged: true
    restart: always
    environment:
    # - K3S_TOKEN=${K3S_PASSWORD_1} # Only required if we are running more than 1 node
    - K3S_KUBECONFIG_OUTPUT=/output/kubeconfig.yaml
    - K3S_KUBECONFIG_MODE=666
    volumes:
    - k3s-server:/var/lib/rancher/k3s
    # This is just so that we get the kubeconfig file out
    - ./k3s_data/kubeconfig:/output
    - ./k3s_data/docker_images:/var/lib/rancher/k3s/agent/images
    expose:
    - "6443"  # Kubernetes API Server
    - "80"    # Ingress controller port 80
    - "443"   # Ingress controller port 443
    ports:
    - 6443:6443
volumes:
  k3s-server: {}
networks:
  default:
    ipam:
      driver: default
      config:
        - subnet: "172.98.0.0/16" # Self-defined subnet on local machine
```

Create the `k3s_data` directory and run the `docker compose up` command:

```bash
mkdir -p k3s_data/kubeconfig
docker compose up -d
```

All that is left is to alias your kubectl to use your K3s kubeconfig, and you can start interacting with your K3s Kubernetes cluster!

```bash
alias k='kubectl --kubeconfig '"${PWD}"'/k3s_data/kubeconfig/kubeconfig.yaml'
k get pods -A

# NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
# kube-system   local-path-provisioner-7b7dc8d6f5-q5ldx   1/1     Running     0          4m23s
# kube-system   coredns-b96499967-kw4gt                   1/1     Running     0          4m23s
# kube-system   metrics-server-668d979685-cvxmv           1/1     Running     0          4m23s
# kube-system   helm-install-traefik-crd-wrjz6            0/1     Completed   0          4m24s
# kube-system   helm-install-traefik-smxtr                0/1     Completed   1          4m24s
# kube-system   svclb-traefik-pxlgs                       2/2     Running     0          3m51s
# kube-system   traefik-7cd4fcff68-lbg4m                  1/1     Running     0          3m51s
```

## RKE2 in Docker

RKE2 is slightly more complicated as there isn't a readily available container image. We will have to build our own container image and publish it to our container registry of choice.

The Dockerfile can be found [here](https://github.com/rancher/rke2/blob/master/Dockerfile). Alternatively, I have already built the container image and you can copy my image.

Once again, the `docker-compose.yml` file:

```YAML
version: '3.7'
services:
  server:
    image: sachua/rke2-test:v1.27.1-rke2r1
    networks:
    - default
    command: server
    tmpfs:
    - /run
    - /var/run
    ulimits:
      nproc: 65535
      nofile:
        soft: 65535
        hard: 65535
    privileged: true
    restart: always
    environment:
   # - RKE2_TOKEN=${RKE2_PASSWORD_1} # Only required if we are running more than 1 node
    - RKE2_KUBECONFIG_OUTPUT=/output/kubeconfig.yaml
    - RKE2_KUBECONFIG_MODE=666
    volumes:
    - rke2-server:/var/lib/rancher/rke2
    # This is just so that we get the kubeconfig file out
    - ./rke2_data/kubeconfig:/output
    - ./rke2_data/docker_images:/var/lib/rancher/rke2/agent/images
    expose:
    - "6443"  # Kubernetes API Server
    - "80"    # Ingress controller port 80
    - "443"   # Ingress controller port 443
    ports:
    - 6443:6443
volumes:
  rke2-server: {}
networks:
  default:
    ipam:
      driver: default
      config:
        - subnet: "172.98.0.0/16"
```

Create the `rke2_data` directory and run the `docker compose up` command:

```bash
mkdir -p rke2_data/kubeconfig
docker compose up -d
```

Finally, alias your kubectl to use your RKE2 kubeconfig, and you can start interacting with your RKE2 Kubernetes cluster!
_RKE2 might take longer to load as it is more resource intensive than K3s_

```bash
alias k='kubectl --kubeconfig '"${PWD}"'/rke2_data/kubeconfig/kubeconfig.yaml'
k get pods -A

# NAMESPACE     NAME                                                   READY   STATUS      RESTARTS        AGE
# kube-system   cloud-controller-manager-7e3475f79d6d                  1/1     Running     1 (2m34s ago)   2m35s
# kube-system   etcd-7e3475f79d6d                                      1/1     Running     0               116s
# kube-system   helm-install-rke2-canal-vznnf                          0/1     Completed   1               2m23s
# kube-system   helm-install-rke2-coredns-7mdnk                        0/1     Completed   0               2m23s
# kube-system   helm-install-rke2-ingress-nginx-xt8s8                  0/1     Completed   0               2m23s
# kube-system   helm-install-rke2-metrics-server-gzjmd                 0/1     Completed   0               2m23s
# kube-system   helm-install-rke2-snapshot-controller-crd-452h4        0/1     Completed   0               2m23s
# kube-system   helm-install-rke2-snapshot-controller-mlfz8            0/1     Completed   2               2m23s
# kube-system   helm-install-rke2-snapshot-validation-webhook-c6zrt    0/1     Completed   0               2m23s
# kube-system   kube-apiserver-7e3475f79d6d                            1/1     Running     0               2m35s
# kube-system   kube-controller-manager-7e3475f79d6d                   1/1     Running     0               2m37s
# kube-system   kube-proxy-7e3475f79d6d                                1/1     Running     0               2m34s
# kube-system   kube-scheduler-7e3475f79d6d                            1/1     Running     0               2m37s
# kube-system   rke2-canal-8ntsg                                       2/2     Running     0               2m2s
# kube-system   rke2-coredns-rke2-coredns-5896cccb79-hngm5             1/1     Running     0               2m3s
# kube-system   rke2-coredns-rke2-coredns-autoscaler-f6766cdc9-d7b2m   1/1     Running     0               2m3s
# kube-system   rke2-ingress-nginx-controller-vvwcp                    1/1     Running     0               48s
# kube-system   rke2-metrics-server-6d45f6cb4d-wl7t8                   1/1     Running     0               72s
# kube-system   rke2-snapshot-controller-7bf6d7bf5f-v9lfb              1/1     Running     0               58s
# kube-system   rke2-snapshot-validation-webhook-b65d46c9f-988vf       1/1     Running     0               70s
```