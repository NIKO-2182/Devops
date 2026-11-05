## Phase 1: Node Preparation (Swap & Storage Cleanup)

Execute these steps across **all nodes** (Master and Worker EC2 instances) to prevent out-of-memory crashes and disk space exhaustion during builds and pod execution.

### 1. Configure Swap Space

Create a 2 GB swap file to give `t3.micro` instances extra memory headway:

```bash
# Allocate 2GB swap file
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make swap persistent across reboots
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab

```

### 2. Clean Docker & System Storage

Prune unused container layers and build caches to free up volume disk space:

```bash
# Clear stopped containers, unused networks, and dangling images
sudo docker system prune -af --volumes

# Clear Docker buildkit cache
sudo docker builder prune -af

```

---

## Phase 2: Build & Push Unified RPC Image (Master Node)

Build a single lightweight Alpine-based Docker image containing both `llama-rpc-server` (for workers) and `llama-cli` (for client execution), then push it to Docker Hub.

### 1. Create `Dockerfile.rpc`

Create the build file on your Master EC2:

```dockerfile
FROM alpine:latest AS builder
RUN apk add --no-cache git cmake make g++ linux-headers
RUN git clone --depth 1 https://github.com/ggml-org/llama.cpp.git /src
WORKDIR /src
RUN cmake -B build -DGGML_RPC=ON -DLLAMA_RPC=ON -DBUILD_SHARED_LIBS=OFF -DLLAMA_BUILD_TESTS=OFF -DGGML_BUILD_TESTS=OFF
RUN cmake --build build --config Release -j$(nproc)
RUN find build/ -type f \( -name "rpc-server" -o -name "llama-rpc-server" -o -name "ggml-rpc-server" \) -exec cp {} /tmp/llama-rpc-server \;
RUN find build/ -type f \( -name "llama-cli" -o -name "main" \) -exec cp {} /tmp/llama-cli \;

FROM alpine:latest
RUN apk add --no-cache libstdc++ libgcc libgomp
COPY --from=builder /tmp/llama-rpc-server /usr/local/bin/llama-rpc-server
COPY --from=builder /tmp/llama-cli /usr/local/bin/llama-cli
EXPOSE 50052
ENTRYPOINT ["llama-rpc-server"]
CMD ["-H", "0.0.0.0", "-p", "50052"]

```

### 2. Build and Push to Docker Hub

Authenticate to Docker Hub and push the image:

```bash
# Log in to Docker Hub
sudo docker login

# Build image with minimal target compilation
sudo docker build -t <Your-Username>/llama-rpc-server:light -f Dockerfile.rpc .

# Push image to registry
sudo docker push <Your-Username>/llama-rpc-server:light

```

---

## Phase 3: Deploy RPC Worker Pods on K3s Cluster

Deploy the RPC worker servers across separate physical nodes using Kubernetes Pod Anti-Affinity rules.

### 1. Create `llama-rpc-workers.yaml`

Create the Kubernetes deployment and service manifest on the Master EC2:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llama-rpc-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: llama-rpc-worker
  template:
    metadata:
      labels:
        app: llama-rpc-worker
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - llama-rpc-worker
              topologyKey: "kubernetes.io/hostname"
      containers:
        - name: rpc-server
          image: <Your-Username>/llama-rpc-server:light
          imagePullPolicy: IfNotPresent
          args: ["-H", "0.0.0.0", "-p", "50052"]
          ports:
            - containerPort: 50052
          resources:
            requests:
              memory: "100Mi"
              cpu: "200m"
            limits:
              memory: "300Mi"
              cpu: "800m"
---
apiVersion: v1
kind: Service
metadata:
  name: llama-rpc-service
spec:
  selector:
    app: llama-rpc-worker
  ports:
    - port: 50052
      targetPort: 50052
  type: ClusterIP

```

### 2. Apply Manifest and Verify Deployment

Deploy the resources and ensure pods are spread across worker nodes:

```bash
# Apply deployment and service
sudo kubectl apply -f llama-rpc-workers.yaml

# Verify pods are Running and note down their IPs
sudo kubectl get pods -l app=llama-rpc-worker -o wide

```

*Expected Output:* Two running pods distributed across different node hostnames, each assigned a cluster IP address.

---

## Phase 4: Download & Verify Model GGUF (Master Node)

Fetch the quantized model binary on the Master EC2 host.

### 1. Download Model File

Download `Qwen2.5-0.5B-Instruct` GGUF (~398 MB) from Hugging Face:

```bash
# Ensure any previous directories or bad files are removed
rm -rf qwen-0.5b.gguf

# Download GGUF binary
wget https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/qwen2.5-0.5b-instruct-q4_k_m.gguf -O qwen-0.5b.gguf

```

### 2. Verify File Integrity

Ensure the file downloaded as a valid GGUF binary:

```bash
# Verify file size (~398MB)
ls -lh qwen-0.5b.gguf

# Check magic header (must output GGUF)
head -c 4 qwen-0.5b.gguf; echo ""

```

---

## Phase 5: Launch Distributed Model Chat (Master Node)

Initiate model execution using the `llama-cli` binary inside your published Docker image, forcing layer offloading across the worker pods.

### 1. Retrieve Active Pod IPs

Fetch the internal pod network IPs for the RPC workers:

```bash
sudo kubectl get pods -l app=llama-rpc-worker -o wide

```

*Take note of `<WORKER_POD_IP_1>` and `<WORKER_POD_IP_2>` from the `IP` column.*

### 2. Run Distributed Model Inference

Launch `llama-cli` on Master using host networking (`--net=host`) to reach the K3s pod network, setting `-ngl 99` to offload model layers:

```bash
sudo docker run -it --rm \
  --net=host \
  -v $(pwd)/qwen-0.5b.gguf:/model.gguf \
  --entrypoint llama-cli \
  <Your-Username>/llama-rpc-server:light \
  -m /model.gguf \
  --rpc <WORKER_POD_IP_1>:50052,<WORKER_POD_IP_2>:50052 \
  -ngl 99 \
  -co auto \
  -sys "You are a helpful AI assistant. Answer concisely in English."

```

---

## Phase 6: Post-Launch Verification

To confirm that model weights are sharded in RAM across worker nodes rather than held locally on Master:

### 1. Inspect Master Startup Logs

Look for tensor offloading initialization lines when `llama-cli` boots:

```text
ggml_rpc_init: added RPC device 0 (<WORKER_POD_IP_1>:50052)
ggml_rpc_init: added RPC device 1 (<WORKER_POD_IP_2>:50052)
llama_model_load: offloading 14 layers to RPC0
llama_model_load: offloading 14 layers to RPC1

```

### 2. Check Live Pod Memory Usage

In a separate terminal window on your Master node during an active chat session:

```bash
# Check worker RAM utilization
sudo kubectl top pods -l app=llama-rpc-worker

# Inspect worker pod buffer allocation logs
sudo kubectl logs -l app=llama-rpc-worker --all-containers=true --tail=20

```

*Verification:* `kubectl top pods` displays active memory allocation (~180 MiB – 220 MiB per pod), and logs confirm active `alloc_buffer` memory calls.



---

## Phase 7: External Web UI & REST API Access via Traefik Ingress

To transition from CLI-based execution to a persistent, web-accessible service, `llama-server` (an OpenAI-compatible HTTP API and Web UI server) was deployed inside K3s and routed externally using K3s’s built-in **Traefik Ingress Controller**.

---

### Architectural & Configuration Adjustments

1. **Binary Compilation Update (`Dockerfile.rpc`):**
Updated the builder stage to compile and bundle the `llama-server` binary alongside `llama-cli` and `llama-rpc-server`.
2. **Kubernetes Command Mapping:**
Used `command: ["/usr/local/bin/llama-server"]` in the container spec (`command` maps to Docker's `ENTRYPOINT`).
3. **Master Node Pinning (`nodeName`):**
Pinned the deployment to `<MASTER_NODE_HOSTNAME>` (`nodeName`) so the pod can directly mount the `hostPath` volume where `qwen-0.5b.gguf` resides.
4. **Ingress Routing:**
Created a Kubernetes Ingress rule using the `traefik` class to forward port `80` HTTP traffic to the internal `llama-server-service` on port `8080`.

---

### Kubernetes Manifests

#### 1. Deployment & ClusterIP Service (`llama-server-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llama-server
  labels:
    app: llama-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: llama-server
  template:
    metadata:
      labels:
        app: llama-server
    spec:
      nodeName: <MASTER_NODE_HOSTNAME>
      containers:
        - name: llama-server
          image: niharkore/llama-rpc-server:light
          imagePullPolicy: Always
          command: ["/usr/local/bin/llama-server"]
          args:
            - "-m"
            - "/model/qwen-0.5b.gguf"
            - "--host"
            - "0.0.0.0"
            - "--port"
            - "8080"
            - "--rpc"
            - "<WORKER_POD_IP_1>:50052,<WORKER_POD_IP_2>:50052"
            - "-ngl"
            - "99"
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: model-volume
              mountPath: /model
      volumes:
        - name: model-volume
          hostPath:
            path: /home/ubuntu
            type: Directory
---
apiVersion: v1
kind: Service
metadata:
  name: llama-server-service
spec:
  selector:
    app: llama-server
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP

```

#### 2. Traefik Ingress Manifest (`llama-ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: llama-server-ingress
  annotations:
    ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: llama-server-service
                port:
                  number: 8080

```

---

### Execution & Verification Commands

#### 1. Apply Manifests & Monitor Pod Status

Deploy both resources on the Master EC2 instance:

```bash
# Apply deployment, service, and ingress
sudo kubectl apply -f llama-server-deployment.yaml
sudo kubectl apply -f llama-ingress.yaml

# Verify pod status and node assignment
sudo kubectl get pods -l app=llama-server -o wide

```

*Verification:* Output confirms `STATUS` is `1/1 Running` and `NODE` matches `<MASTER_NODE_HOSTNAME>`.

#### 2. Confirm Ingress Binding

Verify Traefik has attached the Ingress controller to port 80 across cluster node endpoints:

```bash
sudo kubectl get ingress llama-server-ingress

```

*Verification:* The `ADDRESS` column populates with cluster node IP addresses listening on port `80`.

#### 3. Access Web UI & External OpenAI-Compatible API

* **AWS Security Group Requirement:** Open Port `80` (HTTP) inbound on the Master EC2 instance security group (`0.0.0.0/0`).
* **Web UI Chat Interface:** Open a browser and navigate to:
```text
http://<MASTER_EC2_PUBLIC_IP>/

```


* **REST API Endpoint (`/v1/chat/completions`):** Send a test request from an external terminal:
```bash
curl http://<MASTER_EC2_PUBLIC_IP>/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "What is the capital of Japan?"}
    ]
  }'

```



*Verification:* The browser loads the interactive `llama.cpp` chat web client, and the `curl` request returns an OpenAI-formatted JSON payload with `"content": "Tokyo"`.