# CodeGen

Helm chart for deploying CodeGen service on Red Hat OpenShift with Gaudi accelerator.

## Installing the Chart

To install the chart, run the following:

```console
cd GenAIInfra/helm-charts/
./update_dependency.sh
helm dependency update codegen-openshift-gaudi

export NAMESPACE="insert-your-namespace-here"
export CLUSTERDOMAIN="$(oc get Ingress.config.openshift.io/cluster -o jsonpath='{.spec.domain}' | sed 's/^apps.//')"
export HFTOKEN="insert-your-huggingface-token-here"

# (Optionally) You may customize model and its size:

export MODEL_ID="meta-llama/CodeLlama-7b-hf"
export MODELSIZE="50Gi"

# To run on Xeon
helm install codegen codegen-openshift-gaudi --set image.repository=image-registry.openshift-image-registry.svc:5000/${NAMESPACE}/codegen --set llm-uservice.image.repository=image-registry.openshift-image-registry.svc:5000/${NAMESPACE}/llm-tgi --set react-ui.image.repository=image-registry.openshift-image-registry.svc:5000/${NAMESPACE}/react-ui --set global.clusterDomain=${CLUSTERDOMAIN} --set global.huggingfacehubApiToken=${HFTOKEN}

# (Optionally) Add if you customize model and its size:
--set tgi.model.id=${MODEL_ID} --set tgi.model.size=${MODELSIZE}
```

## Verify

To verify the installation, run the command `oc get pods` to make sure all pods are running. Wait about 5 minutes for building images. When 3 pods achieve *Completed* status, the rest with services should go to *Running*.

## Launch the UI
To access the frontend, find the route for *react-ui* with command `oc get routes` and open it in the browser.
