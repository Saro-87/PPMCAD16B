Helm


We have worked with a lot of K8S manifests, 

If I have to run an app on K8S,

minimum, I would need:

deploy.yaml
     - in dev:
        - may run a different docker image
     - in qa
        - may run a different docker image
service.yaml
     - in dev:
        - may run same service as NodePort
     - in prod
        - it runs as clusterIp with ingress
configmap.yaml
     - in dev:
        - will have different values for DB, REDIS
     - in prod
        - will have different values for DB, REDIS
secrets.yaml
     - in dev:
        - will have different values for DB, REDIS
     - in prod
        - will have different values for DB, REDIS
ingress.yaml
pvc.yaml
statefulsets.yaml

and I used kubectl commands to deploy my application on K8S,

kubectl apply -f <file-names-above>

and you deployed your application on the dev env..

what will you do now, when you have to deploy the same set on QA, UAT, Prod


---

redundant files
sed operator - shell command to replace contents in a file
Helm
