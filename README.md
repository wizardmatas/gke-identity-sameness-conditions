# gke-identity-sameness-conditions

Based on [artice](https://medium.com/google-cloud/solving-the-workload-identity-sameness-with-iam-conditions-c02eba2b0c13) step by step guide how to reproduce conditions based on GKE cluster name


## info

I will use two clusters in different regions `gke-eu-west4` and `gke-eu-west6` ,  `kubectx` will be used to change contexts. 

## Create clusters:

```
gcloud beta container clusters create gke-eu-west4 \
--project ${PROJECT_ID} \
--region europe-west4 \
--cluster-version 1.27.8-gke.1067004 \
--enable-private-nodes \
--enable-ip-alias \
--master-ipv4-cidr 172.16.14.0/28 \
--workload-pool ${PROJECT_ID}.svc.id.goog \
--no-enable-master-authorized-networks \
--shielded-secure-boot \
--shielded-integrity-monitoring \
--async



gcloud beta container clusters create gke-eu-west6 \
--project ${PROJECT_ID} \
--region europe-west6 \
--cluster-version 1.27.8-gke.1067004 \
--enable-private-nodes \
--enable-ip-alias \
--master-ipv4-cidr 172.16.16.0/28 \
--workload-pool ${PROJECT_ID}.svc.id.goog \
--no-enable-master-authorized-networks \
--shielded-secure-boot \
--shielded-integrity-monitoring \
--async
```

## prepare setup

Create bucket and service accunt and give access to service account

```
export PROJECT_ID=matas2222

gsutil mb -l eu -p ${PROJECT_ID} gs://workload-identity-${PROJECT_ID}v2

gcloud iam service-accounts create gsa-wi-sameness2

gsutil iam ch serviceAccount:gsa-wi-sameness2@${PROJECT_ID}.iam.gserviceaccount.com:roles/storage.admin gs://workload-identity-${PROJECT_ID}v2


```

## prepare cluster `gke-eu-west4`

```
kubectx gke-eu-w4

kubectl create namespace test2

kubectl create serviceaccount test2-sa --namespace test2

gcloud iam service-accounts add-iam-policy-binding \
gsa-wi-sameness2@${PROJECT_ID}.iam.gserviceaccount.com \
--role roles/iam.workloadIdentityUser \
--member "serviceAccount:${PROJECT_ID}.svc.id.goog[test2/test2-sa]"

kubectl annotate serviceaccount \
--namespace test2 test2-sa \
iam.gke.io/gcp-service-account=gsa-wi-sameness2@${PROJECT_ID}.iam.gserviceaccount.com

```

## test cluster `gke-eu-west4` (expected result access to GCS bucket)

```
kubectl run -it \
--image google/cloud-sdk:slim \
--namespace test2 \
--overrides='{ "spec": { "serviceAccountName": "test2-sa" } }' \
workload-identity-test


###inside container

root@workload-identity-test:/# gcloud auth list
                  Credentialed Accounts
ACTIVE  ACCOUNT
*       gsa-wi-sameness2@matas2222.iam.gserviceaccount.com

To set the active account, run:
    $ gcloud config set account `ACCOUNT`



root@workload-identity-test:/# touch file-from-west4
root@workload-identity-test:/# export PROJECT_ID=matas2222
root@workload-identity-test:/# gsutil cp file-from-west4  gs://workload-identity-${PROJECT_ID}v2
Copying file://file-from-west4 [Content-Type=application/octet-stream]...
/ [1 files][    0.0 B/    0.0 B]                                                
Operation completed over 1 objects.                                              
root@workload-identity-test:/# gsutil ls gs://workload-identity-${PROJECT_ID}v2
gs://workload-identity-matas2222v2/file-from-west4
root@workload-identity-test:/# 
```


## prepare cluster `gke-eu-west6`


```
kubectx gke-eu-w6

kubectl create namespace test2

kubectl create serviceaccount test2-sa --namespace test2

kubectl annotate serviceaccount \
--namespace test2 test2-sa \
iam.gke.io/gcp-service-account=gsa-wi-sameness2@${PROJECT_ID}.iam.gserviceaccount.com

```

## test cluster `gke-eu-west6` (expected result access to GCS bucket)

```
kubectl run -it \
--image google/cloud-sdk:slim \
--namespace test2 \
--overrides='{ "spec": { "serviceAccountName": "test2-sa" } }' \
workload-identity-test

###inside container
root@workload-identity-test:/# export PROJECT_ID=matas2222
root@workload-identity-test:/# gsutil ls gs://workload-identity-${PROJECT_ID}v2
gs://workload-identity-matas2222v2/file-from-west4
root@workload-identity-test:/# touch file-from-west6
root@workload-identity-test:/# gsutil cp file-from-west6 gs://workload-identity-${PROJECT_ID}v2
Copying file://file-from-west6 [Content-Type=application/octet-stream]...
/ [1 files][    0.0 B/    0.0 B]                                                
Operation completed over 1 objects.                                              
root@workload-identity-test:/# gsutil ls gs://workload-identity-${PROJECT_ID}v2
gs://workload-identity-matas2222v2/file-from-west4
gs://workload-identity-matas2222v2/file-from-west6

```


## conditions 

remove bidning and add biding with condition to allow access only for cluster  `gke-eu-west6`

```
gcloud iam service-accounts remove-iam-policy-binding \
gsa-wi-sameness2@${PROJECT_ID}.iam.gserviceaccount.com \
--role roles/iam.workloadIdentityUser \
--member "serviceAccount:${PROJECT_ID}.svc.id.goog[test2/test2-sa]"

gcloud iam service-accounts add-iam-policy-binding \
gsa-wi-sameness2@${PROJECT_ID}.iam.gserviceaccount.com \
--role roles/iam.workloadIdentityUser \
--member "serviceAccount:${PROJECT_ID}.svc.id.goog[test2/test2-sa]" \
--condition="expression=request.auth.claims.google.providerId=='https://container.googleapis.com/v1/projects/${PROJECT_ID}/locations/europe-west6/clusters/gke-eu-west6',description=single-cluster-acl-6,title=single-cluster-acl-6"

gcloud iam service-accounts get-iam-policy gsa-wi-sameness2@${PROJECT_ID}.iam.gserviceaccount.com
bindings:
- condition:
    description: single-cluster-acl-6
    expression: request.auth.claims.google.providerId=='https://container.googleapis.com/v1/projects/matas2222/locations/europe-west6/clusters/gke-eu-west6'
    title: single-cluster-acl-6
  members:
  - serviceAccount:matas2222.svc.id.goog[test2/test2-sa]
  role: roles/iam.workloadIdentityUser
etag: BwYVQkVfivI=
version: 3

```

## test with conditions  (expected result no access to cluster `gke-eu-west4` and access to cluster `gke-eu-west6` )

```
admin_@cloudshell:~ (matas2222)$ kubectx gke-eu-w4
Switched to context "gke-eu-w4".

admin_@cloudshell:~ (matas2222)$ kubectl run -it \
--image google/cloud-sdk:slim \
--namespace test2 \
--overrides='{ "spec": { "serviceAccountName": "test2-sa" } }' \
workload-identity-test
If you don't see a command prompt, try pressing enter.
root@workload-identity-test:/# export PROJECT_ID=matas2222
root@workload-identity-test:/# gsutil ls gs://workload-identity-${PROJECT_ID}v2
Traceback (most recent call last):
  File "/usr/lib/google-cloud-sdk/platform/gsutil/third_party/apitools/apitools/base/py/credentials_lib.py", line 227, in _GceMetadataRequest
    response = opener.open(request)'
               ^^^^^^^^^^^^^^^^^^^^

exit 
kubectl delete pod workload-identity-test --force --namespace=test2
```

```
kubectx gke-eu-w6
Switched to context "gke-eu-w6".

kubectl run -it \
--image google/cloud-sdk:slim \
--namespace test2 \
--overrides='{ "spec": { "serviceAccountName": "test2-sa" } }' \
workload-identity-test


###inside container
If you don't see a command prompt, try pressing enter.
root@workload-identity-test:/# export PROJECT_ID=matas2222
root@workload-identity-test:/# gsutil ls gs://workload-identity-${PROJECT_ID}v2
gs://workload-identity-matas2222v2/file-from-west4
gs://workload-identity-matas2222v2/file-from-west6
root@workload-identity-test:/

exit 

kubectl delete pod workload-identity-test --force --namespace=test2

```


# add access to cluster `gke-eu-west4` and test (expected result - both clusters have access)

```
gcloud iam service-accounts add-iam-policy-binding \
gsa-wi-sameness2@${PROJECT_ID}.iam.gserviceaccount.com \
--role roles/iam.workloadIdentityUser \
--member "serviceAccount:${PROJECT_ID}.svc.id.goog[test2/test2-sa]" \
--condition="expression=request.auth.claims.google.providerId=='https://container.googleapis.com/v1/projects/${PROJECT_ID}/locations/europe-west4/clusters/gke-eu-west4',description=single-cluster-acl-4,title=single-cluster-acl-4"

Updated IAM policy for serviceAccount [gsa-wi-sameness2@matas2222.iam.gserviceaccount.com].
bindings:
- condition:
    description: single-cluster-acl-4
    expression: request.auth.claims.google.providerId=='https://container.googleapis.com/v1/projects/matas2222/locations/europe-west4/clusters/gke-eu-west4'
    title: single-cluster-acl-4
  members:
  - serviceAccount:matas2222.svc.id.goog[test2/test2-sa]
  role: roles/iam.workloadIdentityUser
- condition:
    description: single-cluster-acl-6
    expression: request.auth.claims.google.providerId=='https://container.googleapis.com/v1/projects/matas2222/locations/europe-west6/clusters/gke-eu-west6'
    title: single-cluster-acl-6
  members:
  - serviceAccount:matas2222.svc.id.goog[test2/test2-sa]
  role: roles/iam.workloadIdentityUser
etag: BwYVQmf1MVE=
version: 3
```

```
kubectx gke-eu-w4
Switched to context "gke-eu-w4".
admin_@cloudshell:~ (matas2222)$ kubectl run -it \
--image google/cloud-sdk:slim \
--namespace test2 \
--overrides='{ "spec": { "serviceAccountName": "test2-sa" } }' \
workload-identity-test

If you don't see a command prompt, try pressing enter.
root@workload-identity-test:/# export PROJECT_ID=matas2222
root@workload-identity-test:/# gsutil ls gs://workload-identity-${PROJECT_ID}v2
gs://workload-identity-matas2222v2/file-from-west4
gs://workload-identity-matas2222v2/file-from-west6
root@workload-identity-test:/#  exit
exit
Session ended, resume using 'kubectl attach workload-identity-test -c workload-identity-test -i -t' command when the pod is running'

kubectl delete pod workload-identity-test --force --namespace=test2
pod "workload-identity-test" force deleted
```

```
kubectx gke-eu-w6
Switched to context "gke-eu-w6".
kubectl run -it \
--image google/cloud-sdk:slim \
--namespace test2 \
--overrides='{ "spec": { "serviceAccountName": "test2-sa" } }' \
workload-identity-test


If you don't see a command prompt, try pressing enter.
root@workload-identity-test:/# export PROJECT_ID=matas2222
root@workload-identity-test:/# gsutil ls gs://workload-identity-${PROJECT_ID}v2
gs://workload-identity-matas2222v2/file-from-west4
gs://workload-identity-matas2222v2/file-from-west6
root@workload-identity-test:/# exit
exit
Session ended, resume using 'kubectl attach workload-identity-test -c workload-identity-test -i -t' command when the pod is running

kubectl delete pod workload-identity-test --force --namespace=test2
Warning: Immediate deletion does not wait for confirmation that the running resource has been terminated. The resource may continue to run on the cluster indefinitely.
pod "workload-identity-test" force deleted'
```



