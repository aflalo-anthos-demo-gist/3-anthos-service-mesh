# Anthos Service Mesh demo guide

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

## Run install script

Cf : `https://cloud.google.com/service-mesh/docs/scripted-install/gke-asm-onboard-1-7`

## Enable Sidecar injection

```
kubectl label namespace default istio-injection- istio.io/rev=asm-173-6 --overwrite
```

## Enable default destination rules

```
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

- Create httpbin v1 and v2 deployments, service and a virtual service to route traffic to v1 only. Also create sleep pod :

```sh
kubectl create -f httpbin-v1.yaml
kubectl create -f httpbin-v2.yaml
kubectl create -f httpbin-svc.yaml
kubectl apply -f httpbin-virtualsvc.yaml
kubectl create -f sleep-deployment.yaml
```

- Send some traffic adn monitor it

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
