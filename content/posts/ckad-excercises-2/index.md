+++
author = "Alexander Liesenfeld"
title = "CKAD Exercises - Part 2"
date = "2021-08-02"
description = "Kubernetes Certified Application Developer (CKAD) Certification Exam Exercises (2021) - Part 2"
tags = [
    "kubernetes",
    "ckad",
    "certification",
]
series = ["CKAD Exercises"]
+++

{{< table_of_contents "Table of Contents">}}

## Pod Design

### Labels, Selectors, Annotations

#### Exercise 1

List all pods with all their labels.

{{< detail-tag "Show Solution" >}}
```bash
kubectl get pods --show-labels
```
{{< / detail-tag >}}

#### Exercise 2

Create two pods with image `nginx`. Add the label `env=blue` to one of them and and `env=green` to the other. List all pods that have the `app` label.

{{< detail-tag "Show Solution" >}}
```bash
kubectl run nginx-blue --image=nginx --restart=Never -l env=blue 
kubectl run nginx-green --image=nginx --restart=Never -l env=green
kubectl get pods -l env --show-labels
```
{{< / detail-tag >}}

#### Exercise 3

Assuming there is a pod with label `env=green`, change the label to `env=red`.

{{< detail-tag "Show Solution" >}}
```bash
kubectl label pod nginx-blue env=red --overwrite=true
```
{{< / detail-tag >}}

#### Exercise 4

Assuming there is a pod with label `env=blue`, remove the value `blue` from this label.

{{< detail-tag "Show Solution" >}}
```bash
kubectl label pod nginx-blue env= --overwrite=true
```
{{< / detail-tag >}}

#### Exercise 5

Assuming there is a pod labeled with `env=blue`, remove this label from the pod.

{{< detail-tag "Show Solution" >}}
```bash
kubectl label pod nginx-blue env-
```
{{< / detail-tag >}}

#### Exercise 6

Add the label `gpu=hardware` to one of your cluster nodes. Create a pod with image `busybox` that will automatically be assigned to run on the previously labeled node.

TIP: Use `nodeSelector` in the pod specification to assign the pod to a node.

{{< detail-tag "Show Solution" >}}
```bash
kubectl get nodes

kubectl label node <one-of-your-nodes-here> gpu=hardware

kubectl run gpu-app --image=busybox --restart=Never --dry-run=client -o yaml > pod.yaml 

kubectl create -f pod.yaml
```

Add a `podSelector` to `pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: gpu-app
  name: gpu-app
spec:
  containers:
  - image: busybox
    name: gpu-app
    resources: {}
  nodeSelector: 
    gpu: hardware
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

> Please note that a `nodeSelector` value is automatically interpreted as a label.
{{< / detail-tag >}}

### Annotations

#### Exercise 7

Annotate a pod with `team=devops`.

{{< detail-tag "Show Solution" >}}
```bash
kubectl annotate pod nginx team=devops
```
{{< / detail-tag >}}

#### Exercise 8

Find a pods annotations.

{{< detail-tag "Show Solution" >}}
```bash
kubectl describe pod nginx
# Please find annotations in the "Annotations" field.
```
{{< / detail-tag >}}

#### Exercise 9

Remove a pods `team=devops` annotation.

{{< detail-tag "Show Solution" >}}
```bash
kubectl annotate pod nginx team-
```
{{< / detail-tag >}}

## Pod Design

### Deployments

#### Exercise 10

Create a deployment with image `nginx`. 

{{< detail-tag "Show Solution" >}}
```bash
kubectl create deploy nginx-webapp --image=nginx:1.20
```
{{< / detail-tag >}}

#### Exercise 11

Get the replicaset that was created with a deployment.

{{< detail-tag "Show Solution" >}}
```bash
kubectl get replicaset -l app=nginx-webapp
```
{{< / detail-tag >}}

#### Exercise 12

Scale an existing deployment to 3 replicas.

{{< detail-tag "Show Solution" >}}
```bash
kubectl scale deploy nginx-webapp --replicas=3 
```
{{< / detail-tag >}}

#### Exercise 13

Find all pods of a deployment.

{{< detail-tag "Show Solution" >}}
```bash
# get the labels from a deployment
kubectl get deploy --show-labels

# use the `app` label to find all deployments
kubectl get pod -l app=nginx-webapp
```
{{< / detail-tag >}}

#### Exercise 14

Get a deployments rollout status.

{{< detail-tag "Show Solution" >}}
```bash
kubectl rollout status deployment nginx-webapp
```
{{< / detail-tag >}}

#### Exercise 15

Change the image version of a deployment. Confirm the image version for the deployment was set correctly. Confirm the pods are started with the new image.

{{< detail-tag "Show Solution" >}}
```bash
kubectl set image deploy/nginx-webapp nginx=nginx:1.19
kubectl get deployment nginx-webapp -o wide # alternatively: kubectl describe deploy nginx-webapp | grep Image
# Confirm that column "IMAGES" contains "nginx:1.19" 
kubectl get pods -l app=nginx-webapp -o yaml | grep "image: nginx:1.19"
# If, e.g. there are 2 pods within the deployment, confirm that the image was listed 2 times.
```

> One alternative to using the "kubectl set image" command is to use the "edit" command and edit the image inside a text editor instead: `kubectl edit deploy nginx-webapp`.
{{< / detail-tag >}}

#### Exercise 16

Show the revision history for a deployment.

{{< detail-tag "Show Solution" >}}
```bash
kubectl rollout history deploy nginx-webapp
```
{{< / detail-tag >}}

#### Exercise 17

Verify that the rollout of the new image version was successful.

{{< detail-tag "Show Solution" >}}
```bash
kubectl rollout history deploy nginx-webapp
kubectl rollout status deployment nginx-webapp
# Make sure that you see a positive confirmation: deployment "nginx-webapp" successfully rolled out.
# If there the deployment is still ongoing or there is a problem, you will usually see a message that makes this clear, 
# such as the following: 
# Waiting for deployment "nginx-webapp" rollout to finish: 1 of 3 updated replicas are available…
# or:
# error: deployment "nginx-webapp" exceeded its progress deadline
```
{{< / detail-tag >}}

#### Exercise 18

Rollback a deployment to the previous version.

{{< detail-tag "Show Solution" >}}
```bash
kubectl rollout undo deployment nginx-webapp
```
{{< / detail-tag >}}

#### Exercise 19 

Set the image of a deployment to a wrong value. Confirm that the deployment is in an error state.

{{< detail-tag "Show Solution" >}}
Use the `set image` command to change the image:

```bash
kubectl set image deploy/nginx-webapp nginx=nginx:does-not-exist
```

TIP: Alternatively, you can also use `kubectl edit deployment nginx-webapp` to edit the image in an editor like `vim`.

Confirm that the rollout is still in a pending state (the corresponding pods should be in status `ErrImagePull`):

```bash
kubectl rollout status deploy nginx-webapp
# Waiting for deployment "nginx-webapp" rollout to finish: 1 old replicas are pending termination...

kubectl get pods -l app=nginx-webapp
# NAME                            READY   STATUS         RESTARTS   AGE
# nginx-webapp-7d85f4f584-v9sqc   0/1     ErrImagePull   0          7s
```
{{< / detail-tag >}}

#### Exercise 20
Suppose there are 10 revisions of a deployment. Rollback all revisions up to revision 1.

{{< detail-tag "Show Solution" >}}
```bash
kubectl rollout undo deployment/nginx-webapp --to-revision=1
```

NOTE: If you leave away parameter `--to-revision` then the `undo` command will only undo the latest revision.
{{< / detail-tag >}}

#### Exercise 21
Suppose there are 10 revisions of a deployment. Show the history for revision 4.

{{< detail-tag "Show Solution" >}}
```bash
kubectl rollout history deploy nginx-webapp --revision=4
# Output:
# deployment.apps/nginx-webapp with revision #4
# Pod Template:
#   Labels:        app=nginx-webapp
#   Containers:
#    nginx:
#     Image:       nginx:1.19
#     Port:        <none>
#     Host Port:   <none>
#     Environment: <none>
#     Mounts:      <none>
#   Volumes:       <none>
```

NOTE: Observe how the image is part of the output!
{{< / detail-tag >}}

#### Exercise 22
Pause the rollout of a deployment.

{{< detail-tag "Show Solution" >}}
```bash
kubectl rollout pause deploy nginx-webapp
```
{{< / detail-tag >}}

#### Exercise 23
Resume the rollout of a deployment.

{{< detail-tag "Show Solution" >}}
```bash
kubectl rollout resume deploy nginx-webapp
```
{{< / detail-tag >}}

#### Exercise 24
Create an autoscaling rule for a deployment. Set the autoscaling CPU target to 70%. Use 2 as the minimum and 3 as the maximum number of replicas.

{{< detail-tag "Show Solution" >}}
```bash
kubectl autoscale deploy webapp --cpu-percent=70 --min=2 --max=3 
```

Make sure a `HorizontalPodAutoscaler` was successfully created.
```bash
kubectl autoscale deploy webapp --cpu-percent=70 --min=2 --max=3 
# Output:
# NAME           REFERENCE                 TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
# nginx-webapp   Deployment/nginx-webapp   <unknown>/70%   2         3         0          7s
```

Verify that two pods have been created:
```bash
kubectl get pods -l app=nginx-webapp
# Output:
# NAME                          READY   STATUS    RESTARTS   AGE
# nginx-webapp-ccf759f6-czv7s   1/1     Running   0          24m
# nginx-webapp-ccf759f6-xr6r2   1/1     Running   0          24m
```
{{< / detail-tag >}}

#### Exercise 25
Remove autoscaling for a deployment.    

{{< detail-tag "Show Solution" >}}
Delete the `HorizontalPodAutoscaler` that was created when the corresponding autoscaling rule has been created.
```bash
kubectl delete hpa nginx-webapp
# Output:
# horizontalpodautoscaler.autoscaling "nginx-webapp" deleted
```
{{< / detail-tag >}}

#### Exercise 26

Delete a deployment.

{{< detail-tag "Show Solution" >}}
```bash
kubectl delete deployment nginx-webapp
```
NOTE: Observe how the pods are being deleted with the deployment as well!
{{< / detail-tag >}}

### Jobs

#### Exercise 27

Create a job named `envprinter` with image `busybox` that prints all environment variables and then quits.

TIP: To print all environment variables, use the linux command `env`.

{{< detail-tag "Show Solution" >}}
```bash
kubectl create job envprinter --image=busybox -- env
# Output: 
# job.batch/envprinter created
```
{{< / detail-tag >}}

#### Exercise 28

Check the status of a job named `envprinter`.

{{< detail-tag "Show Solution" >}}
```bash
kubectl get job envprinter
# Output:
# NAME         COMPLETIONS   DURATION   AGE
# envprinter   1/1           5s         1m14s

kubectl describe job envprinter
# Output: 
# ...
# Type    Reason            Age    From            Message
# ----    ------            ----   ----            -------
# Normal  SuccessfulCreate  4m40s  job-controller  Created pod: envprinter-rn272
# Normal  Completed         4m37s  job-controller  Job completed
```

TIP: You can add parameter `-w` to watch  for changes (such as the completion status).
{{< / detail-tag >}}

#### Exercise 29

Print the logs for a job with the name `envprinter`.

{{< detail-tag "Show Solution" >}}
```bash
kubectl logs job/envprinter
# Output:
# PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
# HOSTNAME=envprinter-crzbq
# KUBERNETES_PORT=tcp://10.245.0.1:443
# KUBERNETES_PORT_443_TCP=tcp://10.245.0.1:443
# KUBERNETES_PORT_443_TCP_PROTO=tcp
# KUBERNETES_PORT_443_TCP_PORT=443
# KUBERNETES_PORT_443_TCP_ADDR=10.245.0.1
# KUBERNETES_SERVICE_HOST=10.245.0.1
# KUBERNETES_SERVICE_PORT=443
# KUBERNETES_SERVICE_PORT_HTTPS=443
# HOME=/root
```

TIP: You can add parameter `-f` to follow the logs. In this case the logs will be streamed.
{{< / detail-tag >}}

#### Exercise 30

Delete the job with the name `envprinter`.

{{< detail-tag "Show Solution" >}}
```bash
kubectl delete job envprinter
# Output:
# job.batch "envprinter" deleted
```
{{< / detail-tag >}}


#### Exercise 31

Create a job named `greeter` with image `busybox` that executes `sh -c 'echo Hi; sleep 3; echo Bye;'` and then exists. Let Kubernetes run it 10 times *in a row* (sequentially). 
Watch the execution progress. Delete the job after all executions completed.

{{< detail-tag "Show Solution" >}}
```bash
kubectl create job greeter --image=busybox --dry-run=client -o yaml -- sh -c 'echo Hi; sleep 3; echo Bye;' > greeter.yaml
```

The above command will create the YAML file `greeter.yaml`. Open this file in a text editor like `vim` and set attribute `spec.parallelism` to value `10` like this: 

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: greeter
spec:
  parallelism: 10 # <--- Add this line to the YAML file!
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command:
        - sh
        - -c
        - echo Hi; sleep 3; echo Bye;
        image: busybox
        name: greeter
        resources: {}
      restartPolicy: Never
status: {}
```

Create the job using the spec from `greeter.yaml`:
```bash
kubectl create -f greeter.yaml 
# Output:
# job.batch/greeter created
```

Watch the progress:
```bash
kubectl get job/greeter -w
# Output:
# NAME      COMPLETIONS   DURATION   AGE
# greeter   0/10          3s         3s
# greeter   1/10          6s         6s
# greeter   2/10          11s        11s
# greeter   3/10          16s        16s
# greeter   4/10          21s        21s
# greeter   5/10          26s        26s
# greeter   6/10          31s        31s
# greeter   7/10          37s        37s
# greeter   8/10          42s        42s
# greeter   9/10          47s        47s
# greeter   10/10         52s        52s
```

IMPORTANT: Observe how we used parameter `-w` to watch for status changes. This command will then block and update the status regularly.

Delete the job:
```bash
kubectl delete job greeter
# Output:
# valuejob.batch "greeter" deleted
```
{{< / detail-tag >}}


#### Exercise 32

Get all pods that belong to a job named `greeter`.

{{< detail-tag "Show Solution" >}}
```bash
kubectl get pods -l job-name=greeter
# Output:
# NAME            READY   STATUS      RESTARTS   AGE
# greeter-4zkm7   0/1     Completed   0          9m8s
# greeter-5hzsv   0/1     Completed   0          9m14s
# greeter-cj22v   0/1     Completed   0          8m56s
# greeter-fq9lk   0/1     Completed   0          9m2s
# greeter-h7xkr   0/1     Completed   0          8m50s
# greeter-jt67x   0/1     Completed   0          9m26s
# greeter-kg7mj   0/1     Completed   0          9m46s
# greeter-qrzv6   0/1     Completed   0          9m20s
# greeter-wm9xx   0/1     Completed   0          9m32s
# greeter-wpxr8   0/1     Completed   0          9m38s
```
{{< / detail-tag >}}

#### Exercise 33

Create a job named `greeter` with image `busybox` that executes `sh -c 'echo Hi; sleep 3; echo Bye;'` and then exists. Let Kubernetes run it 10 times *in parallel*. 
Watch the execution progress. Delete the job after all executions completed.

{{< detail-tag "Show Solution" >}}
```bash
kubectl create job greeter --image=busybox --dry-run=client -o yaml -- sh -c 'echo Hi; sleep 3; echo Bye;' > greeter.yaml
```

The above command will create the YAML file `greeter.yaml`. Open this file in a text editor like `vim` and set attribute `spec.completions` to value `10` like this: 

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: greeter
spec:
  parallelism: 10 # <--- Add this line to the YAML file!
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command:
        - sh
        - -c
        - echo Hi; sleep 3; echo Bye;
        image: busybox
        name: greeter
        resources: {}
      restartPolicy: Never
status: {}
```

Create the job using the spec from `greeter.yaml`:
```bash
kubectl create -f greeter.yaml 
# Output:
# job.batch/greeter created
```

Watch the progress:
```bash
kubectl get job/greeter -w
# Output:
# NAME      COMPLETIONS   DURATION   AGE
# greeter   0/1 of 10     2s         2s
# greeter   1/1 of 10     7s         7s
# greeter   2/1 of 10     8s         8s
# greeter   3/1 of 10     8s         8s
# greeter   4/1 of 10     8s         8s
# greeter   5/1 of 10     8s         8s
# greeter   6/1 of 10     8s         8s
# greeter   7/1 of 10     8s         8s
# greeter   8/1 of 10     12s        12s
# greeter   9/1 of 10     12s        12s
# greeter   10/1 of 10    12s        12s
```

IMPORTANT: Observe how we used parameter `-w` to watch for status changes. This command will then block and update the status regularly.

Delete the job:
```bash
kubectl delete job greeter
# Output:
# valuejob.batch "greeter" deleted
```
{{< / detail-tag >}}


### CronJobs

#### Exercise 34

Create a cron job named `time-printer` that runs each minute to print the current date/time and then exit.
Verify that the job actually runs each minute. Delete the job when you're done.

TIP: You can use the unix command `date` to print the current date/time.

TIP: The cron expression for a 1 minute schedule is `*/1 * * * *`.

{{< detail-tag "Show Solution" >}}

Create the job:
```bash
kubectl create cronjob time-printer --image=busybox --schedule="*/1 * * * *" -- sh -c "date"
# Output:
# cronjob.batch/time-printer created
```

Watch new pods being created each minute:

```bash
kubectl get pods -w
# Output:
# NAME                            READY   STATUS        RESTARTS   AGE
# time-printer-1621548660-7vbgg   0/1     Completed     0          3m8s
# time-printer-1621548720-lgdk4   0/1     Completed     0          2m8s
# time-printer-1621548780-zg6mk   0/1     Completed     0          67s
# time-printer-1621548840-5t76k   0/1     Completed     0          7s
# time-printer-1621548660-7vbgg   0/1     Terminating   0          3m14s
# time-printer-1621548660-7vbgg   0/1     Terminating   0          3m14s
# time-printer-1621548900-wqgb6   0/1     Pending       0          0s
# time-printer-1621548900-wqgb6   0/1     Pending       0          0s
# ...
```

NOTE: Observe how we used parameter `-w` to watch the list of pods. 
This will block and report status updates.

Delete the job: 
```bash
kubectl delete cronjob time-printer
# Output:
# cronjob.batch "time-printer" deleted
```

IMPORTANT: Observe how all pods that were created for the cron job are also being deleted with the cron job!
{{< / detail-tag >}}