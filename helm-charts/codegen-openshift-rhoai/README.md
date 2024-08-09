# CodeGen

Helm chart for deploying CodeGen service on Red Hat OpenShift with Red Hat OpenShift AI.

## Deploy model in Red Hat Openshift AI

1. Log in to the Openshift Container Platform web console and select **Operators** > **OperatorHub**.
2. Search for **Red Hat Openshift AI**, open it and click **Install**.  In the same way install rest of required operators: **Red Hat OpenShift Serverless** and **Red Hat OpenShift Service Mesh**.
3. Open **Red Hat OpenShift AI** dashboard: go to **Networking** -> **Routes** and find location for *rhods-dashboard*).
4. Go to **Data Science Project** and clik **Create data science project**. Fill the **Name** and click **Create**.
5. Go to **Workbenches** tab and clik **Create workbench**. Fill the **Name**, under **Notebook image** choose *Standard Data Science*, under **Cluster storage** choose *Create new persistent storage* and change **Persistent storage size** to 40 GB. Under **Data connections** choose *Create new data connection* and fill all required fields for s3 access. Click **Create workbench**.
6. Open newly created Jupiter notebook and run following commands to download the model and upload it on s3:
```
%env S3_ENDPOINT=<S3_RGW_ROUTE>
%env S3_ACCESS_KEY=<AWS_ACCESS_KEY_ID>
%env S3_SECRET_KEY=<AWS_SECRET_ACCESS_KEY>
%env HF_TOKEN=<PASTE_HUGGINGFACE_TOKEN>
```
```
!pip install huggingface-hub
```
```
import os
import boto3
import botocore
import glob
from huggingface_hub import snapshot_download
bucket_name = 'first.bucket'
s3_endpoint = os.environ.get('S3_ENDPOINT')
s3_accesskey = os.environ.get('S3_ACCESS_KEY')
s3_secretkey = os.environ.get('S3_SECRET_KEY')
path = 'models'
hf_token = os.environ.get('HF_TOKEN')
session = boto3.session.Session()
s3_resource = session.resource('s3',
                               endpoint_url=s3_endpoint,
                               verify=False,
                               aws_access_key_id=s3_accesskey,
                               aws_secret_access_key=s3_secretkey)
bucket = s3_resource.Bucket(bucket_name)
```
```
snapshot_download("ise-uiuc/Magicoder-S-DS-6.7B", cache_dir=f'./models', token=hf_token)
```
```
files = (file for file in glob.glob(f'{path}/**/*', recursive=True) if os.path.isfile(file) and "snapshots" in file)
for filename in files:
    s3_name = filename.replace(path, '')
    print(f'Uploading: {filename} to {path}{s3_name}')
    bucket.upload_file(filename, f'{path}{s3_name}')
```
7. Back to **Red Hat OpenShift AI** dashboard, go to **Settings** -> **Serving runtimes** and click **Add serving runtime**. Choose *Single-model serving platform* and *REST* as API protocol. Upload the file or copy the content of *servingruntime-magicoder.yaml*.

8. Go to your project, then "Models" tab and click **Deploy model** under *Single-model serving platform*. Fill the **Name**, choose newly created **Serving runtime**: *ise-uiuc/Magicoder-S-DS-6.7B on CPU*, **Model framework**: *llm* and change **Model server size** to *Custom*: 16 CPUs and 64 Gi memory. Under **Model location** choose *Existing data connection* and **Path**: *models*. Click **Deploy**. It takes about 10 minutes to get *Loaded* status.

## Install the Chart

To install the chart, run the following:

```console
cd GenAIInfra/helm-charts/
./update_dependency.sh
helm dependency update codegen-openshift-rhoai

export NAMESPACE="insert-your-namespace-here"
export CLUSTERDOMAIN="insert-your-cluster-domain-here"
export HFTOKEN="insert-your-huggingface-token-here"
export MODELNAME="insert-name-of-deployed-model-here"
export PROJECT="insert-project-name-where-model-is-deployed"

# To run on Xeon
helm install codegen codegen-openshift-rhoai --set codegen.image.repository=image-registry.openshift-image-registry.svc:5000/${NAMESPACE}/codegen --set llm-uservice.llmUservice.image.repository=image-registry.openshift-image-registry.svc:5000/${NAMESPACE}/llm-tgi --set codegen-react-ui.reactUi.image.repository=image-registry.openshift-image-registry.svc:5000/${NAMESPACE}/react-ui --set global.clusterDomain=${CLUSTERDOMAIN} --set global.huggingfacehubApiToken=${HFTOKEN} --set llm-uservice.servingRuntime.name=${MODELNAME} --set llm-uservice.servingRuntime.namespace=${PROJECT}
``` 

## Verify

To verify the installation, run the command `oc get pods` to make sure all pods are running. Wait about 5 minutes for building images. When 4 pods achieve *Completed* status, the rest with services should go to *Running*.

## Launch the UI
To access the frontend, find the route for *react-ui* with command `oc get routes` and open it in the browser.
