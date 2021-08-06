+++
author = "Alexander Liesenfeld"
title = "CKAD Exercises - Part 1"
date = "2021-05-16"
description = "Kubernetes Certified Application Developer (CKAD) Certification Exam Exercises (2021) - Part 1"
tags = [
    "kubernetes",
    "ckad",
    "certification",
]
+++

{{< table_of_contents "Table of Contents">}}

## Core Concepts

### Namespaces 

#### Exercise 1

List all namespaces.

{{< detail-tag "Show Solution" >}}
```bash
kubectl get namespaces
```
{{< / detail-tag >}}

#### Exercise 2

Create a new namespace `mynamespace`.

{{< detail-tag "Show Solution" >}}
```bash
kubectl create namespace mynamespace
```
{{< / detail-tag >}}

#### Exercise 3

List all pods in a namespace.

{{< detail-tag "Show Solution" >}}
```bash
kubectl get pods --namespace mynamespace
```
{{< / detail-tag >}}

#### Exercise 4

List all pods in all namespaces.

{{< detail-tag "Show Solution" >}}
```bash
kubectl get pods --all-namespaces
```
{{< / detail-tag >}}

#### Exercise 5

List all deployments and services in all namespaces.

{{< detail-tag "Show Solution" >}}
```bash
kubectl get deployments --all-namespaces
kubectl get services --all-namespaces
```
{{< / detail-tag >}}

### Pod Basics

#### Exercise 6

Create a new pod named `nginx-webapp` into a namespace of your choice other than `default`. Confirm that the pod has started successfully.

{{< detail-tag "Show Solution" >}}
```bash
kubectl run nginx-webapp --image=nginx --restart=Never --namespace mynamespace
kubectl get pod nginx-webapp --namespace mynamespace
## Make sure the pod is in STATUS = Running
## Example:
## NAME        READY   STATUS    RESTARTS   AGE
## nginx-web   1/1     Running   0          2m6s
```

NOTE: Please note that we added `--restart=Never` to make sure that only a regular pod is created. If we would omit this, a deployment would be created that would take care of spinning up our pod. See [here](https://stackoverflow.com/a/40534364) for more information.
{{< / detail-tag >}}

#### Exercise 7

Create a pod `nginx` with image `nginx:1.18-alpine` using a YAML file.

{{< detail-tag "Show Solution" >}}
```bash
kubectl run nginx --image=nginx:1.18-alpine --restart=Never -o yaml --dry-run=client > pod.yaml
kubectl create -f pod.yaml
```
{{< / detail-tag >}}

#### Exercise 8

Get the pod specification for an existing pod in YAML representation.

{{< detail-tag "Show Solution" >}}
```bash
kubectl get pod nginx -o yaml
```
{{< / detail-tag >}}

#### Exercise 9

Get the pod specification for an existing pod in JSON representation.

{{< detail-tag "Show Solution" >}}
```bash
kubectl get pod nginx -o json
```
{{< / detail-tag >}}

#### Exercise 10

Get the details for an existing pod (such as pod status, recent events,  related resources, etc.).

{{< detail-tag "Show Solution" >}}
```bash
kubectl describe pod nginx
```
{{< / detail-tag >}}

#### Exercise 11

Delete a pod.

{{< detail-tag "Show Solution" >}}
```bash
kubectl delete pod nginx
```
{{< / detail-tag >}}

#### Exercise 12

Create a `busybox` pod that executes the shell command `sleep 3600`.

{{< detail-tag "Show Solution" >}}
```bash
kubectl run busybox --image=busybox --restart=Never -- sh -c "sleep 3600"
```
{{< / detail-tag >}}

#### Exercise 13

Create a pod with image `busybox` that only prints 'Hi!' and then stops. Check the logs for the output.

{{< detail-tag "Show Solution" >}}
```bash
kubectl run my-busybox-pod --image=busybox --restart=Never -- sh -c "echo Hi!" 
kubectl logs my-busybox-pod
```
{{< / detail-tag >}}

#### Exercise 14

Connect to a pod and print all available environment variables.

{{< detail-tag "Show Solution" >}}
```bash
kubectl exec nginx -- sh -c env
## or connect to it in interactive mode
kubectl exec nginx -it sh
## print all environment variables using the 'env' command
```
{{< / detail-tag >}}

#### Exercise 15

Create a pod `nginx` with image `nginx:1.18-alpine`. When created, change the image of the pod to `nginx:1.19-alpine`. Convince yourself that the pod has been successfully updated.

{{< detail-tag "Show Solution" >}}
```bash
kubectl set image pod/nginx nginx=nginx:1.19-alpine
kubectl describe pod nginx
## 1: Make sure the Image attribute has been update to nginx:1.19-alpine (see attribute "Containers" > "nginx" > "Image")
## 2: The event log should contain information about a successful image pull and restarting of the pod.
```
{{< / detail-tag >}}

#### Exercise 16

Show the logs of a previously crashed pod.

{{< detail-tag "Show Solution" >}}
```bash
kubectl logs my-busybox-pod -p
```

TIP: This command does not show logs for already terminated pods!
{{< / detail-tag >}}

#### Exercise 17

Show all pod logs from the last hour. 

{{< detail-tag "Show Solution" >}}
```bash
kubectl logs --since=1h nginx
```
{{< / detail-tag >}}

#### Exercise 18

Create a pod with image `nginx` and expose it on port 80. Create another pod with image `busybox` that fetches the index HTML page of the nginx pod and prints it to standard output. Check the logs to see the HTML content.

{{< detail-tag "Show Solution" >}}
```bash
kubectl run nginx-server --image=nginx --restart=Never --port 80
kubectl describe pod nginx-server
## Read the IP of the nginx-server pod from the `describe` command output above.
kubectl run nginx-checker --image=busybox --restart=Never -- sh -c "wget -O- <IP>"
kubectl logs nginx-checker
```
{{< / detail-tag >}}

## Multi-Container Pods

### Multi-Container Pod Basics

#### Exercise 19

Create a multi-container pod with the name `multibox` with two busybox containers that execute the command `sleep 3600`. Check that both containers are running.

{{< detail-tag "Show Solution" >}}
```bash
# Create a single-container pod as usual and print pipe the YAML output into a file
kubectl run multibox --image=busybox --restart=Never --dry-run=client -o yaml -- sh -c "sleep 3600" > multicontainer.yaml
```

Add a second container to the pod specification file we just created. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: multibox
  name: multibox
spec:
  containers:
  - image: busybox
    name: busybox1
    args:
    - sh
    - -c
    - sleep 3600
  - image: busybox
    name: busybox2
    args:
    - sh
    - -c
    - sleep 3600
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

Create the pod:

```bash
kubectl create -f multicontainer.yaml
kubectl get pod multibox
# The READY column in the command output should show 2/2 containers in STATUS = Running. 
```

{{< / detail-tag >}}

#### Exercise 20

Check the logs of one of the containers from a multi-container pod.

{{< detail-tag "Show Solution" >}}
```bash
# Show the logs of container "multibox" of the multi-container pod named "multibox".
kubectl logs multibox -c multibox1
```
{{< / detail-tag >}}

#### Exercise 21

Show the metrics for all pods.

{{< detail-tag "Show Solution" >}}
```bash
kubectl top pods
```
{{< / detail-tag >}}

#### Exercise 22

Show the metrics for all nodes.

{{< detail-tag "Show Solution" >}}
```bash
kubectl top nodes
```
{{< / detail-tag >}}

#### Exercise 23

Create a pod that shares a volume with two containers `nginx` and `busybox`. Mount the volume to path `/usr/share/nginx/html/` in both containers. The `busybox` container must write the current time into file `/usr/share/nginx/html/index.html` every 5 seconds. Once the pod is started, exec into the `nginx` container and verify that the time is being updated every 5 seconds.

TIP: To write the current time into the file, use the shell command `sh -c "while true; do $(date) > /usr/share/nginx/html/index.html; sleep 5; done`

{{< detail-tag "Show Solution" >}}

```bash
kubectl run multicontainer --image=nginx --restart=Never --dry-run=client -o yaml > multicontainer.yaml
```

Add the second container to `multicontainer.yaml` we just created:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: multicontainer
  name: multicontainer
spec:
  volumes:
  - name: html-volume
    emptyDir: {}
  containers:
  - image: nginx
    name: nginx
    resources: {}
    volumeMounts:
    - mountPath: /usr/share/nginx/html/
      name: html-volume
  - image: busybox
    name: busybox
    resources: {}
    volumeMounts:
    - mountPath: /usr/share/nginx/html/
      name: html-volume
    args:
      - sh
      - -c
      - 'while true; do echo "$(date)" > /usr/share/nginx/html/index.html; sleep 5; done'
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

```bash
# verify the pod started successfully
kubectl describe pod multicontainer 
# Exec into the "nginx" container of the "multicontainer" pod 
kubectl exec multicontainer -c nginx -it -- sh
```

Once in the container, execute the following command to show the file content:

```bash
cat /usr/share/nginx/html/index.html 
```
{{< / detail-tag >}}