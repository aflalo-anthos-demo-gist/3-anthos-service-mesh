# Anthos Service Mesh demo gist

## **DISCLAIMER : This repo is a cheat sheet, a lot of steps are missing for a full demo guide**

## Declare necessaries environment variables

```sh
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} \
    --format="value(projectNumber)")
export CLUSTER_NAME=asm-demo
export CLUSTER_ZONE=us-central1-b
export WORKLOAD_POOL=${PROJECT_ID}.svc.id.goog
export MESH_ID="proj-${PROJECT_NUMBER}"
```

## Create a cluster

```sh
gcloud config set compute/zone ${CLUSTER_ZONE}
gcloud beta container clusters create ${CLUSTER_NAME} \
    --machine-type=n1-standard-4 \
    --num-nodes=4 \
    --workload-pool=${WORKLOAD_POOL} \
    --enable-stackdriver-kubernetes \
    --subnetwork=default \
    --release-channel=regular \
    --labels mesh_id=${MESH_ID}
```

## Download installation script for Managed Control Plane

1. Download the version of the script that installs Anthos Service Mesh 1.9.2 to the current working directory:

```sh
curl https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.9 > install_asm
```

2. Download the SHA-256 of the file to the current working directory:

```sh
curl https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.9.sha256 > install_asm.sha256
```

3. With both files in the same directory, verify the download:

```sh
sha256sum -c --ignore-missing install_asm.sha256
```

If the verification is successful, the command outputs: `install_asm: OK` For compatibility, the preceding command includes the `--ignore-missing` flag to allow any version of the script to be renamed to `install_asm`.

4. Make the script executable:

```sh
chmod +x install_asm
```

## Run install script

```sh
./install_asm \
  --project_id PROJECT_ID \
  --cluster_name CLUSTER_NAME \
  --cluster_location CLUSTER_LOCATION \
  --mode install \
  --enable_all
  ```


More installation steps available here : [Installing Anthos Service Mesh](https://cloud.google.com/service-mesh/docs/scripted-install/gke-install)

## Enable Sidecar injection

Before running this command, make sure you're setting ASM version accordingly to your install. More information : [Injecting sidecar proxies](https://cloud.google.com/service-mesh/docs/proxy-injection)

```sh
kubectl label namespace default istio-injection- istio.io/rev=asm-192-1 --overwrite
```


## Enable default destination rules

```sh
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```

## Request routing example

- First we create a virtual service to route all the traffic to the V1

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

- Check subset definitions

```sh
kubectl get destinationrules -o yaml
```

- Restrict users to `jason`for v2

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

## Fault injection

Before starting :

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

We'll create a fault injection only for Jason

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

We injected a delay on rating service of 7s however we do observe that the service is ending in error. Why ? A bug is in the code, there is a hardcoded timeout of 3s

Now we will test to inject an http abort fault

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml
```

As we can see, when connected as Jason the service doesn't answer

## Traffic Shifting

Reset the env : `kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml`

Now let's migrate 50% of the traffic to v3

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

If we consider the service stable we can migrate all the traffic to it :

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
```

We saw here how we can support canary / green blue deployments

## Traffic mirroring

Open a term with four windows

- Create httpbin v1 and v2 deployments, service and a virtual service to route traffic to v1 only. Also create sleep pod :

```sh
kubectl create -f httpbin-v1.yaml
kubectl create -f httpbin-v2.yaml
kubectl create -f httpbin-svc.yaml
kubectl apply -f httpbin-virtualsvc.yaml
kubectl create -f sleep-deployment.yaml
```

- Send some traffic and check logs to see

```sh
#Get Sleep pod and send traffic to app
export SLEEP_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
kubectl exec "${SLEEP_POD}" -c sleep -- curl -s http://httpbin:8000/headers

#Check logs on V1
export V1_POD=$(kubectl get pod -l app=httpbin,version=v1 -o jsonpath={.items..metadata.name})
kubectl logs "$V1_POD" -c httpbin

#Check logs en V2
export V2_POD=$(kubectl get pod -l app=httpbin,version=v2 -o jsonpath={.items..metadata.name})
kubectl logs "$V2_POD" -c httpbin
```

Apply mirror rule : `kubectl apply -f httpbin-mirror-v2.yaml`

- Cleaning up :

```sh
kubectl delete virtualservice httpbin
kubectl delete destinationrule httpbin

#Shutdown the httpbin service and client:

kubectl delete deploy httpbin-v1 httpbin-v2 sleep
kubectl delete svc httpbin
```
