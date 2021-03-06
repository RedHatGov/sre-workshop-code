# OpenShift Site Reliability Engineering Workshop - Code
This repo contains the source code used in a workshop on OpenShift SRE.

For reference the workshop dashboard using this code [can be found here](https://github.com/RedHatGov/sre-workshop-dashboard/).

## Setup

Run the following as cluster admin.

Install the service mesh operators:

```bash
oc create -f ./setup/istio-operators.yaml
```

Wait until all the operators are running:

```bash
oc get pods -n openshift-operators --watch
```

Set the number of users:
```bash
export NUM_USERS=5  # replace me
```

For each user, create a user project and service mesh control plane project:

```bash
for (( i=1 ; i<=$NUM_USERS ; i++ ))
do
  oc new-project user$i --as=user$i \
    --as-group=system:authenticated --as-group=system:authenticated:oauth
  oc new-project user$i-istio --as=user$i \
    --as-group=system:authenticated --as-group=system:authenticated:oauth
done
```

Delete limit ranges in each user project:
> Note: This is only for RHPDS clusters

```bash
for (( i=1 ; i<=$NUM_USERS ; i++ ))
do
  oc delete limitrange --all -n user$i-istio
done
```

Create service mesh control plane in each project:

```bash
for (( i=1 ; i<=$NUM_USERS ; i++ ))
do
  oc create -n user$i-istio -f ./setup/istio-installation.yaml
done
```

Wait until the control plane is running:

```bash
oc get pods -n user$NUM_USERS-istio --watch
```

For each user, add the user project as a member to each user's service mesh project and deploy the microservices application.

```bash
for (( i=1 ; i<=$NUM_USERS ; i++ ))
do
  sed "s|%USER_PROJECT%|user$i|" ./setup/istio-smmr.yaml | oc create -n user$i-istio -f -
  oc new-app -n user$i -f ./setup/microservices-app-ui.yaml -e FAKE_USER=true
  oc new-app -n user$i -f ./setup/microservices-boards.yaml
  oc create -n user$i -f ./setup/gateway.yaml
done
```

Wait until the application pods are running:

```bash
oc get pods -n user$NUM_USERS --watch
```

> Output (sample):

```
NAME                      READY   STATUS      RESTARTS   AGE
app-ui-1-deploy           0/1     Completed   0          22h
app-ui-1-xxxxx            2/2     Running     0          22h
boards-1-xxxxx            2/2     Running     0          22h
boards-1-deploy           0/1     Completed   0          22h
boards-mongodb-1-deploy   0/1     Completed   0          22h
boards-mongodb-1-xxxxx    2/2     Running     0          22h
```

Finally, deploy a quota in each project.
> Note: This is only for the `cronjob` scenario.

```bash
for (( i=1 ; i<=$NUM_USERS ; i++ ))
do
  oc create -n user$i -f ./setup/quota.yaml
done
```

*Note*: The `app-ui` image in Quay was built using [this fork](https://github.com/theckang/service-mesh-workshop-code).  The `boards` image in Quay was built using the [upstream repo](https://github.com/RedHatGov/service-mesh-workshop-code).

## Troubleshooting

1. Envoy does not accept HTTP/1.0 connections by default.  Make sure `ab` can talk to the ingress gateway.

```bash
GATEWAY_URL=$(oc get route istio-ingressgateway -n %username%-istio --template='http://{{.spec.host}}')
curl -I --http1.0 $GATEWAY_URL/stress
ab -n 1 $GATEWAY_URL/stress
```
