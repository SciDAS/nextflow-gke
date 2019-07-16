Nextflow on GKE
===

The major goals of this project is to __automate the execution of Nextflow workflows on GKE__ and __enable interaction with workflows via REST API__.

Currently, it supports the following API endpoints.

| Endpoint                | Method | Description                                 |
|-------------------------|--------|---------------------------------------------|
| /workflow               | GET    | List all workflow instances                 |
| /workflow               | POST   | Create a workflow instance                  |
| /workflow/{id}          | DELETE | Delete a workflow instance                  |
| /workflow/{id}/upload   | POST   | Upload input files to a workflow instance   |
| /workflow/{id}/launch   | POST   | Launch a workflow instance                  |
| /workflow/{id}/status   | GET    | Get the status of a workflow instance       |
| /workflow/{id}/log      | GET    | Get the log of a workflow instance          |
| /workflow/{id}/download | GET    | Download the output data as a `tar.gz` file |

### Lifecycle

First, the user calls the API to create a workflow instance. Along with the API call, the user must provide the __name of the nextflow pipeline__. The payload of the API call is shown below.

```json
{
  "pipeline": "systemsgenetics/KINC-nf"
}
```

Then the user uploads the input and config files for the workflow instance.

After the input and config files in place, the user can launch the workflow. The launch starts with uploading of the input files to `<id>/input` on the NFS service. The jobs running as distributed pods in GKE will read the input data from here, and work together in the dedicated workspace prefixed with `<id>`.

Once the workflow being launched, the log will be available via the API. Ideally, higher-level services can call the API periodically to fetch the latest log of the workflow instance.

After the run is done, the user can call the API to download the output files. The output files are placed in `<id>/output` on the NFS service. The API will compress the directory as a `tar.gz` file for downloading.

The user can call the API to delete the run and purge its data once done with it.

### Deployment

The API is dependent on the following components:

1. kubectl
2. nextflow
3. modified `kube-runner` scripts ([./kube-runner](./kube-runner))
4. Python 3, pip and Tornado
5. gcloud (required for GKE)

The API also assumes that __there is an NFS service running in Kubernetes and shared across the Kubernetes cluster__ and __pods are able to mount and access the NFS storage__. [Here](deploy/README.md) we show how to set up the shared NFS storage in GKE, which is streamlined by Google to certain extent. Additional knowledge about Kubernetes service and NFS may be required to set up the NFS storage in a custom Kubernetes cluster.

#### 0. Quickstart

To deploy the API on a Ubuntu 18.04 LTS node, use the commands below:

```console
$ sudo apt update && sudo apt install -y git python3 python3-pip \
  && git clone https://github.com/SciDAS/nextflow-gke.git \
  && cd nextflow-gke \
  && sudo pip3 install -r requirements.txt \
  && python3 server.py
The API is listening on http://0.0.0.0:8080
```

#### 1. kubectl

The `kuberun` sub-command in Nextflow relies on the `kubectl` to perform any Kubernetes-related activites, e.g., launching pods, mounting storage among many others. In addition, the `kube-runner` scripts also relies on `kubectl` to stage in/out data to/from the shared NFS storage. Following [this tutorial](deploy/README.md), you are able to install and authenticate the `kubectl` to your GKE cluster.

#### 2. nextlfow

[This tutorial](deploy/README.md) has covered how to install the `nextlfow` executable. The API assumes __the nextflow executable is located in any path included in $PATH__.

#### 3. modified kube-runner scripts

First of all, __the API also relies on the `kube-runner` scripts for data staging__, so the scripts in [kube-runner](kube-runner) __must__ be visible in $PATH.

The scripts stem from the [kube-runner](https://github.com/SystemsGenetics/kube-runner) repo. Specifically, we have modified only the `kube-load.sh` and `kube-save.sh` to ensure the data staging behaviors are consistent with the API expects, and in the meantime allow data compression before data stage-out. The scripts may be subject to changes for robustness and stability.

#### 4. Python 3, pip and Tornado

The API is running atop [Tornado](https://www.tornadoweb.org/en/stable/), which is a Python-based, asynchronous web server framework. Specifically, we use the Python 3 version of it. The API is not coupled with any specific feature provided in Tornado, therefore you can use any preferred web server framework as a replacement.
