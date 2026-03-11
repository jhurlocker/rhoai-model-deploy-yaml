# rhoai-model-deploy-yaml
A few YAML files to deploy gpt-oss-20b on OpenShift AI

1. Make sure you are logged into your cluster and switch to your project
oc project <NAMESPACE>

2. Create PVC
oc apply -f pvc.yaml

3. Copy model from quay.io to PVC. Wait for the job to complete successfully. 
oc apply -f copy_model_pvc_job.yaml
oc get job -w

4. add hardware profile for OpenShift AI
oc apply -f hardware_profile.yaml

5. add serving runtime 
oc apply -f serving_runtime.yaml

6. Create inference service
oc apply -f inference_service.yaml

TIPS
1. Create the Inspector Pod
You can pipe this YAML directly into oc apply to create a lightweight UBI Pod that mounts your gpt-oss-20b-pvc claim and stays alive (sleep infinity)

# List the top-level directory
oc exec pvc-inspector -- ls -lh /mnt/pvc/

# Or, open an interactive shell to explore the weights and config files
oc exec -it pvc-inspector -- /bin/sh

oc delete pod pvc-inspector
