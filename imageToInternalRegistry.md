## Objective : Push an image from local system to Openshift Internal Registry

What is an Internal Registry? - Internal registry is a web service deployed in namespace openshift-image-registry.
It has a Kubernetes Service so that traffic can reach it from inside the cluster, but NOT from the internet.

``` oc commands to follow
    $ oc project openshift-image-registry
    $ oc get services 
    
      NAME                  TYPE    CLUSTER-IP   EXTERNAL-IP  PORT(S)   AGE
  image-registry           ClusterIP  172.30.199.92  <none>    5000/TCP  7h17m
  image-registry-operator  ClusterIP  None      <none>    60000/TCP  7h30m
```

Accessing internal registry from one's laptop (local system) is outside the cluster and hence additional steps need to be
taken to access the registry, i.e the service needs to be exposed. 
reference - https://docs.openshift.com/container-platform/4.4/registry/securing-exposing-registry.html#registry-exposing-secure-registry-manually_securing-exposing-registry 

(*To be done only once per cluster*)
``` commands to expose the service
    $ oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
    $ oc get routes
    
    NAME            HOST/PORT                                                                                 PATH      SERVICES         PORT      TERMINATION   WILDCARD
default-route   default-route-openshift-image-registry.apps.dev-svc-4.5-063013.devcluster.openshift.com             image-registry   <all>     reencrypt     None
```
We can start by checking  '$ oc get routes' to see if the image registry service is exposed. If it does, no need to execute the oc patch command.

``` docker login commands
  $ docker login -u <username> -p <password> https://default-route-openshift-image-registry.apps.dev-svc-4.5-063013.devcluster.openshift.com
```

If the login fails with the below error:
```
Error response from daemon: Get https://default-route-openshift-image-registry.apps.dev-svc-4.5-063013.devcluster.openshift.com/v2/: x509: certificate signed by unknown authority
```
Add the registry to daemon.json file (located at /etc/docker in a linux system) as below, and then restart docker for changes to take effect.
``` daemon.json file:
    {
    "debug": true,
    "experimental": true,
    "insecure-registries" : ["default-route-openshift-image-registry.apps.gitops1.devcluster.openshift.com"]
    }
```

Try docker login again, once successfully login, image can be build and tagged using,
``` docker build and tag it
    docker build . -t default-route-openshift-image-registry.apps.dev-svc-4.5-063013.devcluster.openshift.com/NAMESPACE/FOO
    docker push default-route-openshift-image-registry.apps.dev-svc-4.5-063013.devcluster.openshift.com/NAMESPACE/FOO
```
