# rhoai-model-deploy

## Overview

This repository provides a Helm chart to seamlessly deploy generative AI models (defaulting to `gpt-oss-20b`) on Red Hat OpenShift AI (RHOAI). The chart automates the following steps:
1. **Storage Provisioning**: Creates a PersistentVolumeClaim (PVC) to store model weights.
2. **Model Loading**: Executes a Kubernetes Job that copies the model weights from a container image (Modelcar format) directly into the provisioned PVC.
3. **Hardware Configuration**: Creates a Custom Hardware Profile to allocate necessary compute resources (CPUs, Memory, GPUs) for RHOAI.
4. **Model Serving**: Deploys a `ServingRuntime` utilizing the vLLM engine, and an `InferenceService` (KServe) to expose the model as a scalable API endpoint.
5. **Inspection (Optional)**: Deploys a lightweight utility Pod to inspect the downloaded contents of the model PVC.

## Prerequisites

- **OpenShift Container Platform (OCP)** Is installed you are authenticated as admin.
- **NVIDIA GPUs** The Node Feature Discovery and NVIDIA GPU operators are installed and configured on OpenShift.
- **Red Hat OpenShift AI (RHOAI)** Is already installed and configured on the cluster. Admin access is required. 
- **Helm v3+** Is installed on your local machine.

## Installation with Helm

1. Ensure you are logged into your OpenShift cluster and switch to your desired target namespace (project):
   ```bash
   oc project <YOUR_NAMESPACE>
   ```

2. Install the Helm chart:
   ```bash
   helm upgrade --install my-model-release . -f values.yaml
   ```

   *Optional: If you wish to override values such as the model name, PVC size, or GPU counts, you can pass them during installation:*
   ```bash
   helm upgrade --install my-model-release . \
     --set model.name=my-custom-model \
     --set hardwareProfile.gpuCount=2 \
     --set storage.pvcSize=50Gi
   ```

3. Wait for the model copy Job to complete successfully. The model serving components will not be fully ready until the Job finishes copying the weights to the PVC.
   ```bash
   oc get jobs -w
   ```
   Wait until `load-gpt-oss-20b-copy` shows `1/1` completions.

## Configuration

You can customize the deployment by editing the `values.yaml` file. Key configurations include:
- `model.name`: The identifier for your model deployment.
- `model.image`: The Modelcar container image containing the model weights.
- `storage.pvcSize`: Size of the storage needed for the weights (e.g., `30Gi`).
- `hardwareProfile.*`: Resource allocations for model serving, including CPU, memory, and GPU counts.
- `inferenceService.args`: Arguments passed to the vLLM server (e.g., max sequence length, chunking, prefix caching).

## Tips & Troubleshooting

### Using the Inspector Pod
The chart includes an optional inspector pod (enabled by default via `pvcInspector.enabled: true` in `values.yaml`). This pod mounts the PVC and stays alive (`sleep infinity`), allowing you to browse the filesystem and verify the model weights without disrupting the main deployment.

List the files in the PVC:
```bash
oc exec pvc-inspector-gpt-oss-20b -- ls -lh /mnt/pvc/
```

Open an interactive shell inside the PVC inspector:
```bash
oc exec -it pvc-inspector-gpt-oss-20b -- /bin/sh
```

*(Replace `gpt-oss-20b` with your custom model name if you changed it in `values.yaml`.)*

### Testing the Model Endpoint with curl

Once the `InferenceService` is in a ready state, you can test the endpoint using `curl`. The vLLM serving runtime provides an OpenAI-compatible API out of the box.

First, wait for the inference service to be ready and get its external URL:
```bash
oc get inferenceservice gpt-oss-20b
```

Export the URL to a variable (ensure your model name is correct):
```bash
export MODEL_URL=$(oc get inferenceservice gpt-oss-20b -o jsonpath='{.status.url}')
```

Then, send a POST request to the `/v1/chat/completions` endpoint:
```bash
curl -k ${MODEL_URL}/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-oss-20b",
    "messages": [
      {
        "role": "system",
        "content": "You are a helpful assistant."
      },
      {
        "role": "user",
        "content": "What is the capital of France?"
      }
    ],
    "max_tokens": 50
  }'
```

If successful, you should receive a JSON response containing the model's generated text. If you enabled authentication in the `InferenceService` annotations, you'll need to pass an authorization token as well (`-H "Authorization: Bearer <TOKEN>"`).

### Benchmarking with GuideLLM

To get a quick overview of the model's performance (such as throughput and latency), you can use [GuideLLM](https://github.com/NeuralMagic/guidellm). 

Since the vLLM ServingRuntime exposes an OpenAI-compatible API, GuideLLM can directly target your deployment.

1. Install GuideLLM (it is recommended to use a Python virtual environment):
```bash
pip install guidellm
```

2. Run the benchmark tool using the `$MODEL_URL` you exported in the previous section. GuideLLM will test the model under various loads and report the performance metrics:
```bash
guidellm benchmark \
  --target $MODEL_URL/v1 \
  --model gpt-oss-20b \
  --processor RedHatAI/gpt-oss-20b \
  --data "prompt_tokens=256,output_tokens=128" \
  --max-seconds 60 \
  --rate-type concurrent \
  --rate 1,8,32 \
  --outputs json
```

*(Note: If your endpoint uses self-signed certificates or requires authentication, make sure to configure your environment variables or pass the corresponding GuideLLM parameters to handle TLS and API keys appropriately.)*
