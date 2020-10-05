# OpenShift Site Reliability Engineering Workshop - Code
This repo contains the source code used in a workshop on OpenShift SRE.

For reference the workshop dashboard using this code [can be found here](https://github.com/RedHatGov/sre-workshop-dashboard/).

## Setup

Run the following as cluster admin.

Install the service mesh operators:

```bash
oc create -f ./setup/istio-operators.yaml
```

Approve the install plans:
```bash
oc patch ip $(oc get ip -n openshift-operators -o jsonpath={.items[0].metadata.name}) -n openshift-operators --type merge -p '{"spec":{"approved":true}}'
oc patch ip $(oc get ip -n openshift-operators -o jsonpath={.items[1].metadata.name}) -n openshift-operators --type merge -p '{"spec":{"approved":true}}'
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
