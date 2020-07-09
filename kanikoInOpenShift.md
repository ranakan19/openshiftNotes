## Objective: Do a kaniko build in Openshift

### Scenario: source code exists in a github repository. Assumes that Dockerfile is at the root of the repository

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
