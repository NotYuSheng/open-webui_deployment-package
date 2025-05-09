# Open WebUI WebWork (Helm Deployment)

This repository contains a Helm chart for deploying a customized, secured version of [Open WebUI](https://github.com/open-webui/open-webui) with branding, Keycloak authentication, and OpenShift-compatible security.

---

## Loading Image from Tar into OpenShift Internal Registry

> **Note:** This setup assumes your OpenShift environment (e.g., CRC or production) is offline or air-gapped.
> You must load and push the image manually into the OpenShift internal image registry.
> **Ensure your user has push permissions to the internal OpenShift registry.**

### 1. Ensure you're logged in to OpenShift as a user with image push rights

```bash
eval $(crc oc-env)
TOKEN=$(oc whoami -t)
```

### 2. Set the OpenShift internal image registry route

```bash
export REGISTRY=$(oc get route default-route \
  -n openshift-image-registry \
  -o jsonpath='{.spec.host}')
echo "Registry host: $REGISTRY"
```

### 3. Log in to the internal image registry

```bash
podman login \
  --tls-verify=false \
  -u developer \
  -p "$TOKEN" \
  "$REGISTRY"
```

### 4. Load the image from tar file

```bash
podman load -i images/open-webui_webwork.tar
```

### 5. Tag the image for OpenShift internal registry

```bash
podman tag \
  ghcr.io/notyusheng/open-webui_webwork:v1.5.1-6b044964 \
  $REGISTRY/open-webui/open-webui_webwork:v1.5.1-6b044964
```

### 6. Push the image to OpenShift internal registry

```bash
podman push \
  --tls-verify=false \
  $REGISTRY/open-webui/open-webui_webwork:v1.5.1-6b044964
```

---

## Unpack, Configure, Deploy Helm charts

### 1. Extract the chart

```bash
mkdir -p unpacked-chart
tar -xzf ./helm/open-webui-1.0.0.tgz -C unpacked-chart
cd unpacked-chart/open-webui
```

### 2. Edit deployment configuration

Open `values.yaml`:

```bash
nano values.yaml
```

Make the following adjustments:

* **Use the internal image you pushed earlier:**

  ```yaml
  image:
    repository: image-registry.openshift-image-registry.svc:5000/open-webui/open-webui_webwork
    tag: "v1.5.1-6b044964"
    pullPolicy: IfNotPresent
  ```

* **Point to your Keycloak instance running in the same namespace:**

  ```yaml
  # NOTE: This must match the actual DNS/service for Keycloak in your cluster
  keycloak:
    enabled: true
    clientId: "webui-client"
    clientSecret: "your-keycloak-client-secret"
    protocol: "https"
    host: "keycloak.open-webui.svc.cluster.local"
    port: "443"
    realm: "myrealm"
  ```

* **Point to your Keycloak instance running in the same namespace: (Can only be used to configure one instances)**

  ```yaml
  extraEnvVars:
    - name: OPENAI_API_BASE_URL
      value: "http://vllm.open-webui.svc.cluster.local:8000/v1"
  ```

### 3. Deploy the chart

Make sure Keycloak, vLLM, and Open WebUI are all in the same OpenShift project:

```bash
helm install open-webui . -n <namespace>
```

Or to upgrade an existing release:

```bash
helm upgrade open-webui . -n <namespace>
```


---

## Directory Structure

The project structure is as follows:  

```shell
.
├── diagrams
│   ├── c4-diagram.jpg
│   └── c4-diagram_verbose.jpg
├── helm
│   └── open-webui-1.0.0.tgz
├── images
│   └── open-webui_webwork.tar
├── README.md
└── reports
    └── trivy-analysis.txt
```
