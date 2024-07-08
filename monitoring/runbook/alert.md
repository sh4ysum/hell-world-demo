# Monitoring Alerts

This is a sample runbook for resolving a monitoring alert.

## Table of Contents
- [Requirements](#requirements)
- [Architecture](#architecture-and-system-information)
- [Troubleshooting](#troubleshooting)
- [Escalating](#escalating)

## Requirements
- Git
- Docker Desktop
- Kubectl
- Helm

## Architecture and System Information

Sample diagram:

![Sample Architecture](../imgs/prod_env_example.png "Sample Architecture")

## Contributing Systems & Dependencies

List any contributing systems and their dependencies here for potential impact of "hello-world-demo"

Sample list:
- Redis cache
- Dependent backend API service
- Queuing service
- etc.

## Troubleshooting

Ideally you have checked logs or service errors and have pinpointed where the issue may be. If not, review potential monitoring metrics to identify unusual metrics and start your troubleshooting from there.

###  HTTP Errors:

- Potential alerts:
    - HTTP
    - Network

Consider verifying the service and/or the ingress controller. Review pod status if pods are not in a healthy state.

```bash
kubectl get pods -A
```

- Validate logs of NGINX ingress to confirm no service issues with ingress

    ```bash
    kubectl logs -n ingress deploy/ingress-nginx-controller
    ```

- Attempt to restart deployments if no obvious errors are presenting making sure to validate logs and status of pods in between restarts

    ```bash
    kubectl rollout restart -n hello-world deploy/hello-world-demo
    kubectl get pods -n hello-world
    kubectl logs -n hello-world deploy/hello-world-demo
    ```

    ```bash
    kubectl rollout restart -n ingress deploy/ingress-nginx-controller
    kubectl get pods -n ingress
    kubectl logs -n ingress deploy/ingress-nginx-controller
    ```

### Latency or Intermittent Outages:

- Potential alerts:
    - CPU
    - MEMORY
    - I/O

Consider time of day and if it a peak business hour. A new customer or large operational task could be producing extra load on the system. Look for signs of resource contention.

Review metric dashboards and compare to baseline usage to hypothesize if this is a possibility. If so consider mitigating locally and performing a scaling exercise.

```bash
kubectl scale -n hello-world --replicas=3 deployment/hello-world-demo
```

```bash
kubectl get pods -n hello-world
```

Observe if the host is able to support the scaling of the service. If not, consider reviewing auto scale setting for the host and if they were triggered, or if the max scale has been reached.

### Recovery

If all other troubleshooting route have failed, consider rebuilding the service from the latest deployment.

- Uninstall

    ```bash
    helm uninstall -n hello-world hello-world-demo
    ```

- Install
    ```bash
    helm install hello-world-demo hello-world-demo/ --values hello-world-demo/values.yaml --create-namespace --namespace hello-world
    ```

- Validate

    ```bash
    kubectl get pods -A
    ```

    ```bash
    helm test hello-world-demo -n hello-world
    ```

    ```bash
    curl -Lk -H 'Host:hello-world-demo.local' http://localhost
    ```

## Escalating

If you have exhausted your options and still cannot resolve the issue escalate within a timely manner. Consider customer impact and risk of cascading errors/impact.

- Sample escalation path here.

    ```bash
    ├── On-call SRE
    │   ├── App 1
    ├── On-call SE
    │   ├── App 2
    │       └── Backend dependent service
    └── Manager
    ```