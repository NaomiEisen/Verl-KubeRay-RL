# VERL Training on KubeRay

We will use KubeRay for creating VERL training in our OpenShift cluster.

## Setting up the cluster

All the steps above are taken from here:
[KubeRay Operator Helm Chart](https://github.com/ray-project/kuberay/tree/7092f76e6f08fa86ad21c37cd8216914dd215975/helm-chart/kuberay-operator)

I chose the approach of installing the CRDs separately from the operator, to keep the permission requirements for each independent.

### 1. Add the KubeRay Helm repository

```bash
helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm repo update
```

If you're working on the cluster that I am (details omitted for security), then the CRDs are already created in the cluster :))
To see wether the cluster you're working on has them, run:

```bash
oc get crd | grep ray
```

You should see:

```
rayclusters.ray.io
rayjobs.ray.io
rayservices.ray.io
```

If not, the CRDs can be installed with:

```bash
kubectl create -k "github.com/ray-project/kuberay/ray-operator/config/crd?ref=v1.5.1&timeout=90s"
```

But this requires admin.

### 2. Install the operator specifically into your namespace

```bash
helm install kuberay-operator kuberay/kuberay-operator \
  --version 1.5.1 \
  --namespace <your-namespace> \
  --set singleNamespaceInstall=true \
  --skip-crds
```

Note that I used the flags for deploying within the namespace, and I skipped the CRD installation. This will also enable non-admin users to deploy it, in clusters that already have the CRDs.

## Deploying the training

There are different ways to deploy VERL training with KubeRay using its different CRDs. I tried the most basic one: RayCluster.

You can use my YAML configurations to deploy your RayCluster resource (cluster.verl.yaml), just change the namespace to whatever you need. I included the VERL installation and preparation within the YAML configurations.

### Deploy the example

```bash
oc apply -f cluster.verl.yaml
```

Wait for the pod to be created and ready.

### Run the training

```bash
oc exec -ti <pod-name> -- bash
cd /tmp/verl
```
*If you didn't change the name of the deployment, the pod name should start with verl-cluster.*
Validate VERL installation:

```bash
cat /tmp/verl_ready.txt
```

Should see "Setup complete". You can also validate that the VERL code is there (in `/tmp/verl`) and that the data is deployed by running inside the pod:

```bash
ls -lh /tmp/verl/data/gsm8k
```

You should see the two files: `train.parquet` and `test.parquet`.

> If some of the files are missing, just repeat the setup code described in the container `postStart` lifecycle hook in the YAML manually.

## Understanding the thread/CPU configuration (still under investigation)

VERL + vLLM + Ray together spawn many processes and threads (~100+). Each component creates its own threads independently, without coordinating with the others. This leads to a recurring issue that crashes the training loop: **"can't start new thread"** errors.

I don't have a full solution for this yet. My setup for the example workload works, but it may need adjusting for different resource configurations or training parameters.

There are two separate CPU-related settings that control different things:

| Setting | What it controls | Level |
|---|---|---|
| `resources.limits.cpu: "32"` | Max threads/processes the OS allows the container to create (cgroup limit) | OS |
| `rayStartParams.num-cpus: "4"` | How many CPUs Ray *thinks* it has, controlling how many actors it schedules | Application |

We keep `num-cpus` low (4) so Ray only schedules a few concurrent actors. But we keep the CPU limit high (32) because each actor internally spawns many threads (vLLM engine threads, PyTorch threads, async IO, etc.) that Ray has no control over. The OS needs enough cgroup headroom to allow all those threads to be created.

The environment variables like `OMP_NUM_THREADS=1` and `VERL_NUM_THREADS=1` reduce parallelism within each library, but they cannot prevent all thread creation.

### Process accumulation and pod restarts

**Failed or completed VERL training runs leave behind zombie Ray actor processes.** These processes do not fully clean up after the training script exits. Running multiple trainings on the same pod without restarting will accumulate processes until the container hits its thread limit and crashes.

You can check the current process count inside the pod:

```bash
ps aux | wc -l
```

A fresh pod should have ~23 processes. If you see significantly more (100+), the pod needs to be restarted before running another training.

**Always restart the pod between training runs:**

```bash
oc delete raycluster verl-cluster -n verl
oc apply -f cluster.verl.yaml
```

Or from within the pod, run `ray stop --force` (this will trigger a pod restart since Ray is the main process).

You can use a **RayJob** (`job.verl.yaml`) which automatically creates a fresh cluster for each training run and cleans up afterward. This avoids the process accumulation problem entirely.

## Validation caveat

The VERL validation step (`test_freq`) spawns additional threads on top of the already-running training workers (reward scoring, test set generation). This extra thread spike can cause the "can't start new thread" error even on a fresh pod. This is why I set `trainer.test_freq=-1` to skip validation during training.

