# Deployment for Blockify Ingest (v1.0) with NVIDIA NIM

This document contains the setup instructions and Kubernetes manifests for deploying Blockify Ingest, powered by the NVIDIA NIM for Llama 3.1 8B.

---

## 1. Prerequisites: NVIDIA NGC Secrets

Before applying the deployment manifests, you must configure your Kubernetes secrets. The NVIDIA NIM container requires authentication to pull the Docker image and to validate the model license at runtime.

### API Key Permissions
You need an **NVIDIA NGC API Key**. Ensure the key has the following permissions:
1.  **Registry Access:** Read access (to pull `nvcr.io` images).
2.  **Model Access:** Ensure your NGC account has accepted the Terms of Service for the specific model (Llama 3.1) in the NVIDIA NGC Catalog.

### Create the Secrets
Run the following commands to create the necessary secrets. 

> **Note:** Replace `YOUR_NAMESPACE` and ensure your `NGC_API_KEY` environment variable is set in your terminal before running these commands.

#### A. Create the Image Pull Secret
This secret allows Kubernetes to pull the container image from the NVIDIA Container Registry.

```bash
kubectl create secret docker-registry ngc-secret \
  --namespace YOUR_NAMESPACE \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password="${NGC_API_KEY}"
```

#### B. Create the Runtime API Secret
This secret is mounted into the container to authenticate the NIM runtime.

```bash
kubectl create secret generic ngc-api \
  --namespace YOUR_NAMESPACE \
  --from-literal=NGC_API_KEY=${NGC_API_KEY}
```

## 2. Load Model Weights
The NIM container expects the model weights to be present in the persistent storage. Since the model is hosted via a Signed URL, you must populate the volume before the main application can start successfully.

First, apply the Persistent Volume Claim (PVC) from the Kubernetes Manifests section. Then run this one time batch job to download the weights and copy them into storage. Replace the URL with the signed URL provided from Iternal Technologies.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: blockify-model-downloader
spec:
  template:
    spec:
      containers:
      - name: downloader
        image: curlimages/curl:latest
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "Starting download..."
            cd /models
            # Download and extract the model from the signed URL
            curl -L "YOUR_SIGNED_URL_HERE" -o model.tar.gz
            mkdir -p blockify-ingest
            tar -xzf model.tar.gz -C blockify-ingest
            rm model.tar.gz
            echo "Download complete."
        volumeMounts:
        - name: model-storage
          mountPath: /models
      restartPolicy: OnFailure
      volumes:
      - name: model-storage
        persistentVolumeClaim:
          claimName: blockify-model-pvc
```

Run the job and wait for completion.

```bash
kubectl apply -f model-download-job.yaml
kubectl wait --for=condition=complete job/blockify-model-downloader --timeout=300s
kubectl delete -f model-download-job.yaml
```

## Kubernetes Manifests
```yaml
# ===================================================================
# 1. PersistentVolumeClaim (PVC)
# Requests persistent storage for the fine-tuned model weights.
# ===================================================================
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: blockify-model-pvc
spec:
  accessModes:
    # ReadWriteOnce is standard for a single-replica deployment.
    - ReadWriteOnce
  resources:
    requests:
      # 48Gi is a safe allocation for the 8B model.
      storage: 48Gi
  storageClassName: YOUR_STORAGE_CLASS # Replace with the cluster's available StorageClass

---
# ===================================================================
# 2. Deployment
# Defines the stateful workload for the Blockify NIM container.
# ===================================================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blockify-model-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blockify-model
  template:
    metadata:
      labels:
        app: blockify-model
    spec:
      containers:
      - name: blockify-nim-container
        # Standard NVIDIA NIM image for Llama 3 8B Instruct.
        image: nvcr.io/nim/meta/llama-3.1-8b-instruct:latest

        # Configure the NIM container environment.
        env:
        # Tells the NIM to load the fine-tuned model
        # from the mounted storage path instead of the default.
        - name: NIM_MODEL_NAME
          value: "/models/blockify-ingest"
        # Sets the public model name for the API (e.g., /v1/models).
        - name: NIM_SERVED_MODEL_NAME
          value: "blockify-ingest"
        # Provides the NGC API key from a Kubernetes secret.
        - name: NGC_API_KEY
          valueFrom:
            secretKeyRef:
              name: ngc-api # Must match the secret created in step 1B
              key: NGC_API_KEY

        ports:
        # The NIM service runs on port 8000 by default.
        - containerPort: 8000
          name: http-api
        
        resources:
          limits:
            # Request one NVIDIA GPU.
            nvidia.com/gpu: 1

        volumeMounts:
        # Mount the persistent storage for model weights.
        - name: model-storage
          mountPath: /models
        # Mount the high-speed shared memory volume.
        - name: dshm
          mountPath: /dev/shm

      volumes:
      # Define the 'model-storage' volume linked to our PVC.
      - name: model-storage
        persistentVolumeClaim:
          claimName: blockify-model-pvc
      # Define 'dshm' as a 32Gi RAM-backed volume.
      - name: dshm
        emptyDir:
          medium: Memory
          sizeLimit: 32Gi

      # Define the secret needed to pull the image from nvcr.io.
      imagePullSecrets:
      - name: ngc-secret # Must match the secret created in step 1A

---
# ===================================================================
# 3. Service
# Exposes the Deployment internally for testing and applications.
# ===================================================================
apiVersion: v1
kind: Service
metadata:
  name: blockify-model-service
spec:
  type: ClusterIP
  selector:
    app: blockify-model
  ports:
  - protocol: TCP
    # Expose the service on port 80.
    port: 80
    # Forward to port 8000 of container.
    targetPort: 8000
```

