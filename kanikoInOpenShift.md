## Objective: Do a kaniko build in Openshift

### Scenario: source code exists in a github repository. Assumes that Dockerfile is at the root of the repository

#### Push to Dockerhub:
Steps followed:
1. Create a secret (https://github.com/GoogleContainerTools/kaniko/blob/master/docs/tutorial.md#create-a-secret-that-holds-your-authorization-token)
    Since I was using my dockerhub account to push the built image, followed the exact steps as in the link.
2. Create a pod.yaml file:
    ```
      apiVersion: v1
      kind: Pod
      metadata:
        name: kanikogit
      spec:
        containers:
        - name: kaniko
          image: gcr.io/kaniko-project/executor:latest
          args: [
                  "--context=git://github.com/ranakan19/golang-ex.git",
                  "--destination=krana19/kanikogit"]
          volumeMounts:
            - name: kaniko-secret
              mountPath: /kaniko/.docker
        restartPolicy: Never
        volumes:
          - name: kaniko-secret
            secret:
              secretName: regcred
              items:
                - key: .dockerconfigjson
                  path: config.json
    ```
    Note: the default value of --dockerfile flag is Dockerfile within the build context. Thus, no --dockerfile is provided. 
    
    Note: the github repository in the pod.yaml above (git://github.com/ranakan19/golang-ex.git) is forked from git://github.com/openshift/golang-ex.git 
          because the dockerfile had to be modified to remove "USER nobody". User Nobody had to be commented out since was getting permission denied on 
          go build and led to build failure.
          
    Note: able to build and push image without runAsUser: 0 (which we were using initially when building with kaniko in Openshift using local directory)

3. Log into cluster ``` oc login ```
4. Check into your namespace
5. run ```kubectl apply -f pod.yaml``` (if not in the folder containing pod.yaml, provide full path to pod.yaml file)
6. check the creation of pod using ```oc get po``` should display a pod named "kanikogit" and its status
7. If the status is not completed, run ```oc logs kanikogit``` to check logs
8. If the status is completed, the image must be built and pushed to the registry specified in --destination. In my case it was in my repo 
    krana19/kanikogit at dockerhub
9. pod can be deleted if needed using, ```oc delete po kanikogit```

#### Push to openshift internal image registry:

In the previous scenario, the secret created was for dockerhub. When the destination is internal image registry of openshift, the secret needs to be created for access the internal registry instead. 

1. Login to the cluster ```oc login --token=<token> --server=<server>```
2. Check into your namespace
3. Navigate to Secrets, and select the secret which looks like ```builder-dockercfg-xxxx``` reveal/decode values of .dockercfg file associated with the secret.
4. Find the URL of the registry you want to push, (in my case it was image-registry.openshift-image-registry.svc:5000) and copy the value associated with ```auth``` from the json object for that URL.
5. Navigate to .docker folder on your local machine ```cd ~/.docker```
6. Edit config.json file within to append below in ```"auths":```
    ```
    "image-registry.openshift-image-registry.svc:5000": {
		      "auth": "<auth copied in step 4"
		}
      
7. Create secret by running 
```kubectl create secret generic regcred --from-file=.dockerconfigjson=/home/<username>/.docker/config.json  --type=kubernetes.io/dockerconfigjson```
8. Update pod.yaml to look like:
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: kaniko
    spec:
      containers:
      - name: kaniko
        image: gcr.io/kaniko-project/executor:latest
        args: [
                "--context=git://github.com/ranakan19/golang-ex.git",
                "--destination=image-registry.openshift-image-registry.svc:5000/<namespace>/<imagestream name>:<imagestream tag>",
                "--skip-tls-verify"]
        volumeMounts:
          - name: kaniko-secret
            mountPath: /kaniko/.docker
      restartPolicy: Never
      volumes:
        - name: kaniko-secret
          secret:
            secretName: regcred
            items:
              - key: .dockerconfigjson
                path: config.json
 
9. Run ```kubectl apply -f pod.yaml```
10. Check the pod is running, and can check logs
11. Once pod status changes to "complete", From the UI, check the imageStream and ImageStream tags are created.
12. Start a deploy from UI -> +Add -> from container image -> internal image -> select the imagestream and imagestrem tag -> create and the pod should be able to deploy successfully.

