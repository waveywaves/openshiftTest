# Practical Answers

## Question 1

The following commands, when run, would deploy the app using a nodejs s2i builder image, the n expose the service so we would have a route and then we are running curl on that route to see if we are getting the correct output.

```sh
oc new-project q1
oc new-app https://github.com/sclorg/nodejs-ex --name question1
oc expose svc question1
curl $(oc get routes | grep question1 | tr -s ' ' | cut -d ' ' -f 2)
```

## Ouestion 2

For this question which is similar to the last one except the fat that we have to provide the imagestream from within the project. If we check the `nodejs` builder image present in the `openshift` namespace, and run `oc describe is nodejs`. We can see that the location or the docker image is given which would be the `nodeshift/centos7-s2i-nodejs` tagged image in the docker.io registry.

```sh
oc new-project q2
```

Allow the `developer` user to push images to the Openshift internal Registry.

```sh
oc login -u system:admin
oc adm policy add-role-to-user system:registry developer
oc adm policy add-role-to-user system:image-builder developer
```

Add the openshift internal registry as an insecure registry in docker.

```sh
oc project default
oc expose svc docker-registry
export OPENSHIFTREGISTRY=$(oc get route | grep docker | tr -s ' ' | cut -d ' ' -f 2):80
```

Add `--insecure-registry=$OPENSHIFTREGISTRY` option to the /etc/sysconfig/docker configuration file under the OPTIONS line and restart the docker service.

```sh
systemctl restart docker
```

Login to the Openshift internal registry using docker.

```sh
oc login -u developer
export TOKEN=$(oc whoami -t)
docker login -u developer -p $TOKEN $OPENSHIFTREGISTRY 
```

Pull and tag the image in the local docker registry and then push the image to the Openshift Registry.

```sh
docker pull nodeshift/centos7-s2i-nodejs
docker tag docker.io/nodeshift/centos7-s2i-nodejs $OPENSHIFTREGISTRY/q2/nodenew
docker push $OPENSHIFTREGISTRY/q2/nodenew
```

Check if an imagestream for nodenew is created in the q2 project

```sh
oc login -u developer
oc project q2
oc get is | grep nodenew
```

Deploy the project and check.

```sh
oc new-app nodenew~https://github.com/sclorg/nodejs-ex --name question2
oc expose svc question2
curl $(oc get routes | grep question2 | tr -s ' ' | cut -d ' ' -f 2)
```
----------------------------------------------------------------------------------

### Create ConfigMap from a file

Here we would be injecting a configmap file as a volume

```sh
echo PHRASE="Ask and it will be given to you; search, and you will find; knock and the door will be opened for you." > question2config.configmap 

oc create configmap question2config --from-file ./question2config.configmap
oc set volume dc/question2 --add -t configmap -m /opt/configmap/config --name q2configVol --configmap-name q2config
oc exec question2-5-qph48 cat /opt/configmap/config/question2config.configmap
```

### Create Secrets and inject them as Environment Variables

Here we would be injecting a Secret to be used as environment variables.

```sh
oc create secret generic --from-literal username=vibhav --from-literal password=123
oc set env dc question2 --from secret/question2secret
```

## Question 3

#### Note : Answers for the questions worked well when I tried them out with `oc debug -f "Name of the file"`. But, when I ran `oc create -f "Name of the file"`, the pods would go into CrashLoopBackoff State. 

Due to the new concept presented in this question I had to refer online and followed [this link](https://medium.com/@jmarhee/using-initcontainers-to-pre-populate-volume-data-in-kubernetes-99f628cd4519).

### Create a pod artifact with an initContainer which should provide a file in a volume, usable in the main application container.

```
oc new-project question3
 
docker pull alpine
docker tag docker.io/alpine $OPENSHIFTREGISTRY/question3/alpine
docker push $OPENSHIFTREGISTRY/question3/alpine
```

```
oc create -f question3c1.yaml
```

The init container is going to populate the volume `vol` which is of type `emptyDir`. Then this same volume is mounted by the application pod.

### Create a pod with a non-persistent volume 

An `emptyDir` type non persistent volume has been mounted on the pod.

```sh
oc create -f question3c2.yaml
```

### Create a pod and use NodePort service to expose it

For this question we are going to create a simple deployment with an `httpd` pod and then expose the deployment in using a service of type `NodePort`.

```sh
oc create -f question3c3.yaml
oc expose deployment question3-3 --type=NodePort
export NODEPORT=$(oc describe svc question3-3 | grep NodePort: | tr -s ' ' | cut -d  ' ' -f 3 | cut -d '/' -f 1)
```

Test if the service works properly.
```
curl http://$(minishift ip):$NODEPORT
```

## Question 4 

Deploy cakephp-mysql-persistent project. I completed this task by choosing the template from the Add-to-project menu in the Openshift webconsole. 

The sql pod didn't start but i noticed that the pv was bound so I waited for the cake php build to get completed, after that I perfored a `oc rollout mysql`, after which `mysql` was deployed successfully. 

After checking the logs in the `cakephp` pods it was obvious that it was not able to connect to the `mysql` pod. So, I ran the `oc start-build cakephp-mysql-persistent` command so that there would be a rebuild and after that the pod was built and deployed successfully.

The running pods have been exported as yamls to be checked as `question4*.yaml` files.

## Question 5

As this question contained a new concept I had to refer from online resources. [1](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)

A job is when a batch of tasks are run by one or more pods and these pods terminate upon the completion of that job.

```sh
oc create -f question5jobs.yaml
```

The above artifact upon running starts a job which just `echo`es something, We can see that the requirement of the job is to 4 completions to be done with a maximum of 4 pods running in parallel.

# Theory Answers

#### What is the difference between a pod and a deployment ?
A pod is an instance of a deployment where a deployment contains the specifications for a pod and which containers it should contain and lot of other things like the secrets being used, configmaps ets.

#### How build config and S2I work in short ?

A build config provides a base for an application image to be instantiated with some funtionality like triggers which can allow the application to stay up to date. 

S2I is one kind of buildconfig which allows for converting a given source code directly to an image without having to manually convert the source to a docker formatted image. Builder images are used for this purpose and other methods by which we can understand what kind of source code is provided.

#### What is an imagestream ?

An imagestream can be seen as a historic representation of images over time where the latest image sits on top of the imagestream stack.

#### What is liveliness probe, readiness probe ? When would you use either ?

Both of them are used for health monitoring for containers in pods.
Liveliness probe can allow for the application to be up for most times wven though the application might be buggy.
Readiness probe is used for seeing if the certain container is in a state to be exposed to the outside world.