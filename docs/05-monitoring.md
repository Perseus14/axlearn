# GCP Monitoring

**This doc is still under construction.**

This document describes how to enable GCP workload monitoring.

## Overview

Google Cloud currently lacks visibility into user workload performance. To address this, for critical workloads sensitive to infrastructure changes, Google Cloud offers a customer workload performance monitoring feature. This allows Google engineers to closely track workload performance metrics. If the performance falls below a specified threshold, the Google Cloud on-call team will be alerted to initiate an investigation.

To enable this feature, users must opt-in by publishing a workload performance indicator and a workload heartbeat to the Cloud Monitoring platform using the specified metrics. After publishing, users should collaborate with their Customer Engineer (CE) and the Google team to define appropriate alert thresholds for the performance metrics.

<br>

### Pre-requisites

We assume you have:
1. GCP account and GKE cluster setup.
2. Service account with access to Storage Bucket, Artifact Repository and GKE.
3. Have atleast 2 nodepools with 4x4 tpu-v6e machines in the cluster.

<br>

### Instructions

```bash
#Copy and edit the default config file.
cp .axlearn/axlearn.default.config .axlearn/.axlearn.config
```
For monitoring, make sure the following are present.

```bash
project="my-gcp-project"
zone="my-zone"
labels="my-unique-label"
enable_gcp_workload_monitoring = true
workload_id = "my_workload_id"
replica_id = "0"
```

Build a Docker image using the `Dockerfile.tpu` in the repo:
```bash
docker build -f Dockerfile.tpu --target tpu -t <zone>-docker.pkg.dev/<project-id>/<repo>/tpu:latest .
docker push <zone>-docker.pkg.dev/<project-id>/<repo>/tpu:latest
```

Create a Kubernetes secret to provide access to your GCS bucket:
```bash
kubectl create secret generic gcs-key --from-file=</path/to/your-service-account-key>.json
```

Next, we create a GKE Job yaml file, which defines the cluster, nodepools, topology etc:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: axlearn-headless-svc
  namespace: default
spec:
  clusterIP: None
  selector:
    job-name: axlearn-workload-job
---
apiVersion: batch/v1
kind: Job
metadata:
  name: axlearn-workload-job
  namespace: default
spec:
  completionMode: Indexed  # Required for TPU workloads
  backoffLimit: 0
  completions: 4
  parallelism: 4
  template:
    metadata:
      labels:
        job-name: axlearn-workload-job
    spec:
      restartPolicy: Never
      subdomain: axlearn-headless-svc
      tolerations:
      - key: "google.com/tpu"
        operator: "Exists"
        effect: "NoSchedule"
      nodeSelector:
        cloud.google.com/gke-tpu-accelerator: tpu-v6e-slice
        cloud.google.com/gke-tpu-topology: 4x4
      dnsPolicy: ClusterFirstWithHostNet  # Ensure proper name resolution for TPU pods
      containers:
      - name: test-container
        image: <zone>-docker.pkg.dev/<project-id>/<repo>/tpu:latest
        ports:
        - containerPort: 8471  # Default TPU communication port
        - containerPort: 9431  # TPU metrics port for monitoring
        command:
        - /bin/bash
        - -c
        - |
          echo "Job starting!";
          axlearn gcp config activate --label=my-unique-label;
          python3 -m axlearn.common.launch_trainer_main --module=text.gpt.c4_trainer --config=fuji-7B-v2-flash --trainer_dir=gs://<train-dir>-axlearn/<train-dir>-gke-v6e-7b/ --data_dir=gs://axlearn-public/tensorflow_datasets --jax_backend=tpu --mesh_selector=tpu-v6e-16 --trace_at_steps=16
          echo "Job completed!";

        env:
        - name: JAX_PLATFORMS
          value: "tpu"  # Let JAX auto-detect TPU
        - name: JAX_USE_PJRT_C_API_ON_TPU
          value: "1"
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: "/secrets/your-service-account-key.json"  # Path to the mounted service account key
        volumeMounts:
        - name: gcs-key
          mountPath: "/secrets"
          readOnly: true
        resources:
          requests:
            google.com/tpu: "4"  # Adjust based on TPU topology
          limits:
            google.com/tpu: "4"
      volumes:
      - name: gcs-key
        secret:
          secretName: gcs-key
```

Please modify the yaml file as needed.

Finally, launch the GKE service:
```bash
kubectl apply -f <job-name>.yaml
```

(Optional) Multi-Slice Workload Observability

Create a Jobset yaml file <jobset>.yaml

```yaml
apiVersion: jobset.x-k8s.io/v1alpha2
kind: JobSet
metadata:
  name: axlearn-multislice-jobset
  namespace: default
  annotations:
    alpha.jobset.sigs.k8s.io/exclusive-topology: cloud.google.com/gke-nodepool
spec:
  failurePolicy:
    maxRestarts: 4
  replicatedJobs:
    - name: axlearn-multislice-job
      replicas: 2  # Each slice corresponds to a replicated job
      template:
        spec:
          completionMode: Indexed  # Required for TPU workloads
          backoffLimit: 0
          completions: 4
          parallelism: 4
          template:
            metadata:
              labels:
                job-name: axlearn-multislice-job
            spec:
              restartPolicy: Never
              subdomain: axlearn-headless-svc
              tolerations:
              - key: "google.com/tpu"
                operator: "Exists"
                effect: "NoSchedule"
              nodeSelector:
                cloud.google.com/gke-tpu-accelerator: tpu-v6e-slice
                cloud.google.com/gke-tpu-topology: 4x4
              dnsPolicy: ClusterFirstWithHostNet  # Ensure proper name resolution for TPU pods
              hostNetwork: true
              containers:
              - name: test-container
                image: <zone>-docker.pkg.dev/<project-id>/<repo>/tpu:latest
                ports:
                - containerPort: 8471  # Default TPU communication port
                - containerPort: 8431  # TPU metrics port for monitoring
                - containerPort: 2112  # Additional port for TPU monitoring
                command:
                - /bin/bash
                - -c
                - |
                  env
                  echo "Job starting!";
                  python3 -m axlearn.common.launch_trainer_main --module=text.gpt.c4_trainer --config=fuji-7B-v2-flash --trainer_dir=gs://<your-bucket-name>/<dir-name>-gke-v6e-7b/ --data_dir=gs://axlearn-public/tensorflow_datasets --jax_backend=tpu --mesh_selector=tpu-v6e-16 --trace_at_steps=16
                  echo "Job completed!";
                env:
                - name: JAX_PLATFORMS
                  value: "tpu"  # Let JAX auto-detect TPU
                - name: JAX_USE_PJRT_C_API_ON_TPU
                  value: "1"
                - name: GOOGLE_APPLICATION_CREDENTIALS
                  value: "/secrets/<your-service-account-key>.json"  # Path to the mounted service account key
                volumeMounts:
                - name: gcs-key
                  mountPath: "/secrets"
                  readOnly: true
                resources:
                  requests:
                    google.com/tpu: 4
                  limits:
                    google.com/tpu: 4
              volumes:
              - name: gcs-key
                secret:
                  secretName: gcs-key
```

Apply, the jobset on the GKE cluster:
```bash
kubectl apply -f <jobset>.yaml
```
