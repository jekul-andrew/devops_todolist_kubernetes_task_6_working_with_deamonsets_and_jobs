# Kubernetes DaemonSet and CronJob Instructions

## 1. Overview

This Django ToDo application is deployed to a Kubernetes cluster in the `mateapp` namespace.

The main application is exposed through a ClusterIP service and a NodePort service.

The NodePort service is available in a browser at:

`http://localhost:30007`

This task also adds:

- a `DaemonSet` that continuously checks the application through the ClusterIP service;
- a `CronJob` that calls the `/api/health` endpoint every 4 minutes;
- CPU and memory resource requests and limits for both workloads.

The communication flow is:

```text
DaemonSet / CronJob
        ↓
ClusterIP Service
        ↓
ToDo application Pods
```

## 2. Create and Select the Namespace

Create the namespace:

```bash
kubectl apply -f namespace.yml
```

Set `mateapp` as the default namespace for the current Kubernetes context:

```bash
kubectl config set-context --current --namespace=mateapp
```

After this, Kubernetes commands can be executed without specifying `-n mateapp`.

## 3. Deploy the Application

Apply the application Deployment:

```bash
kubectl apply -f deployment.yml
```

Apply the ClusterIP service:

```bash
kubectl apply -f clusterIp.yml
```

Apply the NodePort service:

```bash
kubectl apply -f nodePort.yml
```

If the project contains an HPA manifest, apply it as well:

```bash
kubectl apply -f hpa.yml
```

Check the current state:

```bash
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get hpa
```

## 4. Validate the Application in a Browser

The application is exposed through the NodePort service on port `30007`.

Open:

```text
http://localhost:30007
```

The application can also be tested through port forwarding:

```bash
kubectl port-forward service/todoapp-service 8080:80
```

Then open:

```text
http://localhost:8080
```

The health endpoint can be checked at:

```text
http://localhost:8080/api/health
```

## 5. Deploy the DaemonSet

Apply the DaemonSet manifest:

```bash
kubectl apply -f daemonset.yml
```

Check the DaemonSet:

```bash
kubectl get daemonset
```

Check the Pods created by the DaemonSet:

```bash
kubectl get pods -l app=todo-daemon -o wide
```

A DaemonSet normally creates one Pod on each eligible Kubernetes node.

The DaemonSet repeatedly executes a request to the ClusterIP service:

```text
curl → sleep 5 seconds → curl → sleep 5 seconds → ...
```

This continuously verifies access to the ToDo application from inside the cluster.

## 6. DaemonSet Resource Requests and Limits

The DaemonSet container uses:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "100Mi"
  limits:
    cpu: "200m"
    memory: "200Mi"
```

`requests.cpu: 100m` means that Kubernetes reserves approximately 0.1 CPU core for the container.

`requests.memory: 100Mi` means that the scheduler takes 100 MiB of memory into account when placing the Pod on a node.

`limits.cpu: 200m` restricts the container to approximately 0.2 CPU core.

`limits.memory: 200Mi` limits the maximum memory usage to 200 MiB.

These values are suitable for this workload because the container only performs a small `curl` request every 5 seconds and does not require significant CPU or memory. The limits are higher than the requests to provide some temporary resource headroom.

## 7. Validate the DaemonSet with Logs

Show logs from Pods created by the DaemonSet:

```bash
kubectl logs -l app=todo-daemon
```

Follow the logs in real time:

```bash
kubectl logs -l app=todo-daemon -f
```

The logs should show repeated responses from the ToDo application approximately every 5 seconds.

## 8. Deploy the CronJob

Apply the CronJob manifest:

```bash
kubectl apply -f cronjob.yml
```

Check the CronJob:

```bash
kubectl get cronjobs
```

After the CronJob has executed, check created Jobs:

```bash
kubectl get jobs
```

Check the Pods created by Jobs:

```bash
kubectl get pods
```

Completed CronJob Pods normally have the status:

```text
Completed
```

## 9. CronJob Configuration

The CronJob uses the following configuration:

```yaml
schedule: "*/4 * * * *"
concurrencyPolicy: Allow
successfulJobsHistoryLimit: 10
failedJobsHistoryLimit: 5
```

### Schedule

```yaml
schedule: "*/4 * * * *"
```

This means that the CronJob runs every 4 minutes.

Each run calls the ToDo application health endpoint through the ClusterIP service:

```text
http://todoapp-service.mateapp.svc.cluster.local/api/health
```

### Concurrency Policy

```yaml
concurrencyPolicy: Allow
```

This allows Kubernetes to start a new Job even if the previous Job has not finished yet.

For this task, each Job performs only one short HTTP request, so the Jobs are expected to finish quickly.

### Successful Job History

```yaml
successfulJobsHistoryLimit: 10
```

Kubernetes keeps the 10 most recent successfully completed Jobs.

### Failed Job History

```yaml
failedJobsHistoryLimit: 5
```

Kubernetes keeps the 5 most recent failed Jobs.

These history limits allow previous executions to be inspected without keeping an unlimited number of completed Jobs.

## 10. CronJob Resource Requests and Limits

The CronJob container uses:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "100Mi"
  limits:
    cpu: "200m"
    memory: "200Mi"
```

The CronJob performs only one short `curl` request to the `/api/health` endpoint, so it does not require significant CPU or memory.

The `requests` values define the resources considered by the scheduler when placing the Job Pod.

The `limits` values prevent the container from consuming excessive CPU or memory while still providing more resources than the minimum requested values.

## 11. CronJob Restart Policy

The CronJob Pod template uses:

```yaml
restartPolicy: OnFailure
```

This means that if the container terminates with an error, Kubernetes may restart the container as part of the same Job.

If the command completes successfully, the container is not restarted and the Job finishes successfully.

## 12. Validate the CronJob with Logs

First, list created Jobs:

```bash
kubectl get jobs
```

For example:

```text
todo-cronjob-29843696
```

Logs can be viewed directly through the Job:

```bash
kubectl logs job/todo-cronjob-29843696
```

Alternatively, find the Pod created by the Job:

```bash
kubectl get pods -l job-name=todo-cronjob-29843696
```

Then inspect its logs:

```bash
kubectl logs <pod-name>
```

The logs should contain the response from:

```text
/api/health
```

## 13. Final Verification

Check all important Kubernetes resources:

```bash
kubectl get pods
kubectl get services
kubectl get daemonset
kubectl get cronjobs
kubectl get jobs
```

Expected result:

- ToDo application Pods are running;
- the ClusterIP and NodePort services are available;
- the application opens in a browser through `http://localhost:30007`;
- DaemonSet Pods are running on eligible nodes;
- DaemonSet logs show repeated requests to the application;
- the CronJob creates Jobs every 4 minutes;
- successful CronJob Pods finish with `Completed` status;
- CronJob logs contain the response from `/api/health`.
