Here's the `README.md` file content, formatted for easy copy-pasting directly into your GitHub repository.

-----

# Helm Kubernetes Volume Guide

This repository provides a comprehensive guide and Helm chart for setting up persistent storage and NGINX Ingress with `k3d` Kubernetes clusters. It's designed to help you understand and implement volume management for your containerized applications efficiently.

-----

## Table of Contents

  * [Features](https://www.google.com/search?q=%23features)
  * [Workflow Overview](https://www.google.com/search?q=%23workflow-overview)
  * [Prerequisites](https://www.google.com/search?q=%23prerequisites)
  * [Getting Started](https://www.google.com/search?q=%23getting-started)
      * [Step 1: Prepare the Host and Cluster](https://www.google.com/search?q=%23step-1-prepare-the-host-and-cluster)
      * [Step 2: Set up Ingress](https://www.google.com/search?q=%23step-2-set-up-ingress)
      * [Step 3: Configure and Deploy the Helm Chart](https://www.google.com/search?q=%23step-3-configure-and-deploy-the-helm-chart)
  * [Verification](https://www.google.com/search?q=%23verification)
  * [Troubleshooting](https://www.google.com/search?q=%23troubleshooting)
  * [Cleanup](https://www.google.com/search?q=%23cleanup)
  * [Contributing](https://www.google.com/search?q=%23contributing)
  * [License](https://www.google.com/search?q=%23license)

-----

## Features

  * **Persistent Storage:** Learn how to configure and use host path volumes for persistent data.
  * **NGINX Ingress:** Set up `ingress-nginx` to manage external access to your applications.
  * **`k3d` Integration:** Utilize `k3d` for lightweight, local Kubernetes cluster development.
  * **Helm Chart:** A ready-to-use Helm chart for deploying a sample web application with configured persistence and ingress.

-----

## Workflow Overview

The guide follows a clear, step-by-step process to get your persistent web application running:

1.  **Prepare Host & Cluster:** Set up your local storage directory and create a `k3d` cluster with the host path mounted.
2.  **Set Up Ingress:** Deploy the `ingress-nginx` controller to manage incoming traffic.
3.  **Deploy Helm Chart:** Configure and install your custom Helm chart with defined persistent volumes and ingress rules.
4.  **Verify Deployment:** Confirm that your application is running, accessing persistent data, and reachable via ingress.

<!-- end list -->

```
+-------------------+     +-----------------+     +-----------------+     +-----------------+
| 1. Prepare Host & | --> | 2. Setup Ingress| --> | 3. Deploy Helm  | --> | 4. Verify       |
|      Cluster      |     |                 |     |      Chart      |     |    Deployment   |
+-------------------+     +-----------------+     +-----------------+     +-----------------+
```

-----

## Prerequisites

Before you begin, ensure you have the following tools installed on your system:

  * **Docker:** Used by `k3d` to run Kubernetes nodes as containers.
      * [Install Docker](https://docs.docker.com/get-docker/)
  * **`k3d`:** A lightweight wrapper to run `k3s` (Rancher Lab's minimal Kubernetes distribution) in docker.
      * [Install k3d](https://www.google.com/search?q=https://k3d.io/%23installation)
  * **`helm`:** The Kubernetes package manager.
      * [Install Helm](https://helm.sh/docs/intro/install/)
  * **`kubectl`:** The Kubernetes command-line tool.
      * [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

-----

## Getting Started

Follow these steps to set up your environment and deploy the sample web application.

### Step 1: Prepare the Host and Cluster

Create a local directory for persistent data and initialize your `k3d` cluster.

```bash
# Create the storage directory and a sample index.html
sudo mkdir -p /mnt/data
sudo chmod -R 777 /mnt/data
echo "<h1>Welcome to My Web App</h1>" | sudo tee /mnt/data/index.html

# Create the k3d cluster with volume mount and disabled Traefik
k3d cluster create mycluster \
  --servers 1 \
  --agents 1 \
  --k3s-arg "--disable=traefik@server:0" \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer" \
  --volume /mnt/data:/mnt/data@all
```

### Step 2: Set up Ingress

Install the `ingress-nginx` controller. Remember to replace `34.29.60.206` with your actual external IP if different.

```bash
# Add and update the ingress-nginx Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install ingress-nginx with your external IP
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.externalIPs[0]=34.29.60.206
```

### Step 3: Configure and Deploy the Helm Chart

Create your Helm chart and configure its `values.yaml` and templates for persistence and ingress.

1.  **Create the Helm Chart:**

    ```bash
    helm create mywebapp
    ```

2.  **Define a `StorageClass`:**
    Create a file named `manual-storageclass.yaml` in your working directory:

    ```yaml
    # manual-storageclass.yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: manual
    provisioner: rancher.io/local-path
    volumeBindingMode: WaitForFirstConsumer
    reclaimPolicy: Delete
    ```

    Apply it to your cluster:

    ```bash
    kubectl apply -f manual-storageclass.yaml
    ```

3.  **Update `mywebapp/values.yaml`:**
    Modify the `mywebapp/values.yaml` file to include the following configurations for replicas, image, service, configMap, persistence, and ingress:

    ```yaml
    # mywebapp/values.yaml
    replicaCount: 2

    image:
      repository: nginx
      pullPolicy: IfNotPresent
      tag: "latest"

    service:
      name: myweb-mywebapp
      type: ClusterIP
      port: 80

    configMap:
      enabled: true
      data:
        default.conf: |
          server {
            listen 80;
            server_name localhost;
            location / {
              root /usr/share/nginx/html;
              index index.html;
            }
          }

    persistence:
      enabled: true
      storageClass: manual
      accessMode: ReadWriteOnce
      size: 1Gi
      path: /mnt/data

    ingress:
      enabled: true
      className: nginx
      annotations: {}
      hosts:
        - host: mywebapp.local
          paths:
            - path: /
              pathType: Prefix
    ```

4.  **Define Helm Chart Templates:**
    Ensure the following files exist and contain the specified content within your `mywebapp/templates/` directory:

      * `mywebapp/templates/pv.yaml`:

        ```yaml
        {{- if .Values.persistence.enabled -}}
        apiVersion: v1
        kind: PersistentVolume
        metadata:
          name: {{ .Release.Name }}-pv
        spec:
          storageClassName: {{ .Values.persistence.storageClass }}
          capacity:
            storage: {{ .Values.persistence.size }}
          accessModes:
            - {{ .Values.persistence.accessMode }}
          hostPath:
            path: {{ .Values.persistence.path }}
        {{- end -}}
        ```

      * `mywebapp/templates/pvc.yaml`:

        ```yaml
        {{- if and .Values.persistence.enabled (not .Values.persistence.existingClaim) }}
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: {{ .Release.Name }}-pvc
        spec:
          storageClassName: {{ .Values.persistence.storageClass | default "" }}
          accessModes:
            - {{ .Values.persistence.accessMode }}
          resources:
            requests:
              storage: {{ .Values.persistence.size }}
        {{- end }}
        ```

      * `mywebapp/templates/deployment.yaml`:

        ```yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: {{ .Release.Name }}-deployment
          labels:
            {{- include "mywebapp.labels" . | nindent 4 }}
        spec:
          replicas: {{ .Values.replicaCount }}
          selector:
            matchLabels:
              {{- include "mywebapp.selectorLabels" . | nindent 8 }}
          template:
            metadata:
              labels:
                {{- include "mywebapp.selectorLabels" . | nindent 12 }}
            spec:
              containers:
                - name: {{ .Chart.Name }}
                  image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
                  imagePullPolicy: {{ .Values.image.pullPolicy }}
                  ports:
                    - name: http
                      containerPort: 80
                      protocol: TCP
                  volumeMounts:
                    - name: config-volume
                      mountPath: /etc/nginx/conf.d/
                      # subPath: default.conf
                    - name: data-volume
                      mountPath: /usr/share/nginx/html
              volumes:
                - name: config-volume
                  configMap:
                    name: {{ .Release.Name }}-configmap
                - name: data-volume
                  persistentVolumeClaim:
                    claimName: {{ if .Values.persistence.existingClaim }}{{ .Values.persistence.existingClaim }}{{ else }}{{ .Release.Name }}-pvc{{ end }}
        ```

      * `mywebapp/templates/configmap.yaml`:

        ```yaml
        {{- if .Values.configMap.enabled -}}
        apiVersion: v1
        kind: ConfigMap
        metadata:
          name: {{ .Release.Name }}-configmap
        data:
          default.conf: |
              server {
                listen 80;
                server_name localhost;
                location / {
                  root /usr/share/nginx/html;
                  index index.html;
                }
              }
        {{- end -}}
        ```

5.  **Install the Helm Chart:**
    Navigate to the `mywebapp` directory and install the chart:

    ```bash
    cd mywebapp
    helm install myweb . --namespace webapp --create-namespace
    ```

-----

## Verification

Confirm your deployment is successful:

1.  **Check Pods:**

    ```bash
    kubectl get pods -n webapp
    ```

    (Ensure your `myweb-deployment` pods are running.)

2.  **Verify `index.html` File:**
    Replace `<YOUR_POD_NAME>` with the actual name of one of your `myweb-deployment` pods from the previous step.

    ```bash
    kubectl exec -it -n webapp <YOUR_POD_NAME> -- ls -la /usr/share/nginx/html
    ```

    (You should see `index.html` listed.)

3.  **Test Access via `curl`:**
    First, add an entry to your `/etc/hosts` file (replace `34.29.60.206` with your LoadBalancer's external IP if different):

    ```bash
    # nano /etc/hosts
    34.29.60.206 mywebapp.local
    ```

    Then, access your web app:

    ```bash
    curl http://mywebapp.local
    ```

    You should see:

    ```html
    <h1>Welcome to My Web App</h1>
    ```

-----

## Troubleshooting

  * **Ingress Controller Not Ready:** If `ingress-nginx` pods are not running or the service doesn't get an external IP, check logs:
    `kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx`
    Ensure your `--set controller.service.externalIPs[0]=<YOUR_IP>` is correct for your environment.
  * **Pod Pending/CrashLoopBackOff:** If your `myweb-deployment` pods are stuck, check pod descriptions and logs:
    `kubectl describe pod <YOUR_POD_NAME> -n webapp`
    `kubectl logs <YOUR_POD_NAME> -n webapp`
    This often indicates issues with volume mounts, permissions, or NGINX configuration.
  * **`curl` Fails:** Double-check your `/etc/hosts` entry and ensure the IP matches the `k3d` load balancer IP. Also, verify the NGINX configuration in the ConfigMap.

-----

## Cleanup

To remove the cluster and all deployed resources:

```bash
k3d cluster delete mycluster
sudo rm -rf /mnt/data
```

-----

## Contributing

Contributions are welcome\! If you find any issues or have suggestions for improvements, please open an issue or submit a pull request.

-----

## License

This project is open-sourced under the MIT License.
