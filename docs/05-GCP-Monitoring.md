## ML Goodput Monitoring
AXLearn supports automatic measurement and upload of workload metrics such as Goodput, Badput Breakdown and Step Time Deviation using the ML Goodput Measurement library.

The [ML Goodput Measurement](https://github.com/AI-Hypercomputer/ml-goodput-measurement) library currently supports monitoring workloads running on Google Cloud Platform. For more information on details of the library, visit the Github page or the [ml-goodput-measurement](https://pypi.org/project/ml-goodput-measurement/) PyPI package documentation.

### What is Goodput
Goodput is the metric that measures the efficiency of model training jobs, i.e. productive time spent on training progress proportional to the total time spent by the workload. It is an actionable way for users to monitor where they can improve to get the most value from their accelerators. 

### What is Badput
Badput is the metric that measures time that a workload spent on anything that is not productive training proportional to the total time spent by the workload. For example, the time spent in accelerator initialization, training preparation, program startup, data loading, portions of checkpointing, disruptions and wasted progress since the last checkpoint etc. all contribute to Badput. 

The ML Goodput Measurement library exposes Badput Breakdown. Further details of each bucket can be found [here](https://github.com/AI-Hypercomputer/ml-goodput-measurement?tab=readme-ov-file#badput-breakdown-details)

### What is Step Time Deviation

Step Time Deviation is the metric that measures deviation of step time from ideal step time.

The ML Goodput Measurement library exposes step time deviation by computing ideal step time or allowing users to configure ideal step time.


### Prerequisites
The usage of this package requires the setup of a Google Cloud project with
billing enabled to properly use Google Cloud Logging. If you don't have a Google
Cloud project, or if you don't have billing enabled for your Google Cloud
project, then do the following:

1. In the Google Cloud console, on the project selector page,
 [select or create a Google Cloud project](https://cloud.google.com/resource-manager/docs/creating-managing-projects).

2. Make sure that billing is enabled for your Google Cloud project. Instructions can be found [here](https://cloud.google.com/billing/docs/how-to/verify-billing-enabled#console)

3. Enable the [Cloud Logging API]((https://console.cloud.google.com/flows/enableapi?apiid=logging.googleapis.com&_ga=2.27841276.1571868865.1726250448-123998259.1726107009) ).

4. To run your training on Cloud accelerator, set up the environment by following instructions [here](https://cloud.google.com/tpu/docs/setup-gcp-account).

5. To learn more about Google Cloud Logging, visit this [page](https://cloud.google.com/logging/docs).

### Access Scopes

 You will need both read and write access scopes for cloud logging on both the
 GPU or TPU and CPU node pools. Full cloud logging access is granted by the
 following access scope during node pool creation:

  - `https://www.googleapis.com/auth/cloud-platform`

   > **_NOTE:_** Access Scopes are immutable and workloads can only be migrated
  to new node pools with required access scopes. Access scopes on already created clusters cannot be updated.

### How to Monitor Goodput and Badput

To enable Goodput recording and monitoring on AXLearn, follow the example below. 

```bash
   axlearn gcp gke start --instance_type=tpu-v5litepod-16 \
        --bundler_type=artifactregistry --bundler_spec=image=tpu \
        --bundler_spec=dockerfile=Dockerfile \
        --name=<jobset-name> \
        -- python3 -m ...training-config... \
        --recorder_type=axlearn.cloud.gcp.measurement:goodput \
        --recorder_spec=name=<jobset-name> \
        --recorder_spec=upload_dir=<my-output-directory>/summaries \
        --recorder_spec=upload_interval=30 \
```

### How to Monitor Step Time Deviation

AXLearn enables step time deviation monitoring by default. You can configure the upload frequency by setting `--recorder_spec=step_deviation_interval_seconds=30`. To disable step deviation set `--recorder_spec=include_step_deviation=False`.

```bash
   axlearn gcp gke start --instance_type=tpu-v5litepod-16 \
        --bundler_type=artifactregistry --bundler_spec=image=tpu \
        --bundler_spec=dockerfile=Dockerfile \
        -- python3 -m my_training_job \
        --recorder_type=axlearn.cloud.gcp.measurement:goodput \
        --recorder_spec=name=my-run-with-goodput \
        --recorder_spec=upload_dir=my-output-directory/summaries \
        --recorder_spec=upload_interval=30 \
        --recorder_spec=include_step_deviation=True \
        --recorder_spec=step_deviation_interval_seconds=30 \
```

### Visualize on Tensorboard

1. Requires packages: `tensorboard-plugin-profile`, `tensorflow` and `tensorboard`.
2. Use the Tensorboard URL on AXLearn logs to view all metrics in one location.

### Enabling Google Cloud Monitoring

AXLearn has an additional option of pushing goodput, badput and step time deviation metrics to Google Cloud Monitoring. By default if goodput monitoring is enabled, the data starts getting uploaded to Google Cloud Monitoring. Set `--recorder_spec=enable_gcp_goodput_metrics=False` and  `--recorder_spec=enable_gcp_step_deviation_metrics=False` to disable goodput and step_deviation uploads to GCM respectively.

```bash
   axlearn gcp gke start --instance_type=tpu-v5litepod-16 \
        --bundler_type=artifactregistry --bundler_spec=image=tpu \
        --bundler_spec=dockerfile=Dockerfile \
        -- python3 -m my_training_job \
        --recorder_type=axlearn.cloud.gcp.measurement:goodput \
        --recorder_spec=name=my-run-with-goodput \
        --recorder_spec=upload_dir=my-output-directory/summaries \
        --recorder_spec=upload_interval=30 \
        --recorder_spec=include_step_deviation=True \
        --recorder_spec=step_deviation_interval_seconds=30 \
        --recorder_spec=enable_gcp_goodput_metrics=True \
        --recorder_spec=enable_gcp_step_deviation_metrics=True
```

#### Visualization in Google Cloud Monitoring

To visualize the collected metrics within Google Cloud Monitoring:

1.  Verify that the workload is executing with monitoring enabled. This ensures automatic data ingestion into Google Cloud Monitoring.
2.  Navigate to [Metrics Explorer](https://console.cloud.google.com/monitoring/metrics-explorer). Initiate metric selection by clicking "Select a metric," then search for and select the "Workload" resource. Subsequently, choose the "Workload" metric category.

    a.  [**Productive Time:**](https://cloud.google.com/monitoring/api/metrics_gcp#:~:text=workload/goodput_time) Represents the cumulative duration the workload spent on productive tasks, measured by `compute.googleapis.com/workload/goodput_time`.  
    b.  [**Non-Productive Time:**](https://cloud.google.com/monitoring/api/metrics_gcp#:~:text=workload/badput_time) Represents the cumulative duration the workload spent on non-productive tasks, measured by `compute.googleapis.com/workload/badput_time`.  
    c.  [**Performance:**](https://cloud.google.com/monitoring/api/metrics_gcp#:~:text=workload/performance) Represents the workload's performance metric, specifically step deviation in this context, measured by `compute.googleapis.com/workload/performance`.