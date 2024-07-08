# "Hello World" Demo

## Table of Contents
- [Context](#context)
- [Requirements](#requirements)
- [Getting Started](#getting-started)
- [Updating](#updating)
- [Rollback](#rollback)
- [Troubleshooting](#troubleshooting)
- [Monitoring](monitoring/README.md)

## Context
This demo is focused around deploying a basic `Hello World` web server using Docker, Kubernetes and Helm.

This NGINX `Hello World` web server will be deployed using Docker Desktop with Kubernetes enabled, an NGINX ingress controller with load balancing enabled, and HELM for the deployment of the service manifests.

There are some general assumptions being made with this demo:
- You are comfortable using the CLI
- You are familiar with Docker, Helm and Kubernetes
- You will need access to the `/etc/hosts` file if you would like to access endpoints via a browser.

## Requirements
- Git
- Docker Desktop
- Kubectl
- Helm
- Editor of choice

## Getting Started
The `hello-world-demo` Helm chart was made using steps provided in [the wiki section](wiki/helm-chart-build.md) if you would like to compose files locally, but it is suggested to clone the repo so no extra unknowns are added.

- Clone the repo and navigate to the hello
- Insure Docker Desktop is running and Kubernetes is enabled
- Update & Build the Dependencies

  ```bash
  helm dependency update
  ```
  ```bash
  helm dependency build
  ```

- Deploy From Local Chart ( this is done from the main directory ).

  ```bash
  helm install hello-world-demo hello-world-demo/ --values hello-world-demo/values.yaml --create-namespace --namespace hello-world
  ```

- Validate
  
  `Example: Hello, World!`
  ```bash
  curl -ILk -H 'Host:hello-world-demo.local' http://localhost
  ```

  `Example reply:`

  ```bash
  $ curl -Lk -H 'Host:hello-world-demo.local' http://localhost
  <html>
      <head>
          <title>Hello World Demo</title>
      </head>
      <body>
          <h1>Hello, World!</h1>
      </body>
  ```

  `Example: Health Endpoint`

  ```bash
  curl -ILk -H 'Host:hello-world-demo.local' http://localhost/health
  ```

  `Example reply:`

  ```bash
  $ curl -Lk -H 'Host:hello-world-demo.local' http://localhost/health
  I am healthy.
  ```

  - You may also update your `/etc/hosts` file to use local resolution so you may see the `"Hello, World!"` text in a browser.

  `Example:`

  ```bash
  127.0.0.1	hello-world-demo.local
  ```

## Updating

Any updates needed should be applied with a new `Chart.yaml` `version` and tested with the `--dry-run` option so that you may confirm configs prior to deployment. Consider using the `--cleanup-on-fail` flag as well if there is suspicion of a failure to avoid manual clean up resources.

### An Example: Update the NGINX chart version.
Update the dependency NGINX to a newer chart `4.10.1` and build the new chart.

- Update NGINX dependency chart version in `Chart.yaml`

  ```bash
  dependencies:
    - name: ingress-nginx
      version: "4.10.1"
      repository: https://kubernetes.github.io/ingress-nginx
  ```

- Update & build the new NGINX dependency

  ```bash
  helm dependency update
  ```
  ```bash
  helm dependency build
  ```
  You should see the `ingress-nginx-4.10.0.tgz` get updated to `ingress-nginx-4.10.1.tgz` if done correctly.

- Increase the `version` value in `Chart.yaml` using Semantic Versioning. Let's say this is a patch.

  ```bash
  version: 0.1.1
  ```

- Deploy the updated chart.
Because we did not update anything within the `values.yaml` we will reuse the values already provided.

  ```bash
  helm upgrade --reuse-values hello-world-demo hello-world-demo/ -n hello-world
  ```

  - If you did modify the `values.yaml` file, please use the `--values` flag with the `values.yaml` file.

    ```bash
    helm upgrade --values hello-world-demo/values.yaml hello-world-demo hello-world-demo/ -n hello-world
    ```

- Validate the site is working
  ```bash
  helm test hello-world-demo -n hello-world
  ```

  ```bash
  curl -ILk -H 'Host:hello-world-demo.local' http://localhost
  ```

## Rollback

### An Example: Rollback the "hello-world-demo" chart.

Let's say there was an issue with a release and we know it was working previously, so let's roll back to a known working chart version.

- Grab the release history for the `hello-world-demo` chart in the `hello-world` namespace

  ```bash
  helm history hello-world-demo -n hello-world
  ```

- Select the revision you would like to rollback to
  
  ```bash
  helm rollback hello-world-demo 1 -n hello-world
  ```

- Validate the site is working
  ```bash
  curl -ILk -H 'Host:hello-world-demo.local' http://localhost
  ```

## Troubleshooting
Troubleshooting ideally starts from an error so we have a general idea of where to start. I will surmise some scenarios one might face during this demo.

### Deployment Issues
See what the error may suggest if output is given from a command. However, potential syntax errors are not always clear.

`Example: Missing namespace`

Without specifying a namespace, default is usually where deployments go.

```bash
$ helm upgrade hello-world-demo hello-world-demo/
Error: UPGRADE FAILED: "hello-world-demo" has no deployed releases
```

`Try:`

```bash
helm upgrade hello-world-demo hello-world-demo/ -n hello-world
```

If you are having issues deploying the chart, let's start with a lint to see if there is a syntax issue. Lint within the chart directory

```bash
helm lint
```

Okay, linting looks good, but it still fails. Does a `--dry-run` succeed? 

`Example: Upgrade with --dry-run added`

```bash
helm upgrade --reuse-values hello-world-demo hello-world-demo/ --namespace hello-world --dry-run
  ```

### Serving Content Related

I get an error when trying to curl, or view in the browser, our localhost deployment.

- Is the webserver pod running and in a healthy state?
  ```bash
  kubectl get pods -n hello-world
  ```

#### Potential Steps
- Let's describe the pod(s) that are unhealthy and see why.
  ```bash
  kubectl describe pod -n hello-world <pod-name-here> 
  ```
- Logs look good, but it still doesn't work. Let's restart the deployment.
  ```bash
  kubectl rollout restart -n hello-world deploy/hello-world-demo
  ```

- Let's review configurations for the service then the ingress controller.
  
  ```bash
  kubectl describe svc -n hello-world hello-world-demo
  ```
  `From the output:`
  ```
    Type:              ClusterIP
    IP Family Policy:  SingleStack
    IP Families:       IPv4
    IP:                10.105.179.245
    IPs:               10.105.179.245
    Port:              http  80/TCP
    TargetPort:        80/TCP
    Endpoints:         10.1.0.41:8080,10.1.0.42:8080
  ```
  `Expected:`
  ```bash
  TargetPort:        8080/TCP
  ```

  Remember, the `hello-world` image we used is listening over port `8080`. Did you include the `targetPort: 8080` line in `values.yaml` ?


- Verify the chart files and confirm there are no issues and try a fresh deployment of the chart.

  `Uninstall:`
  ```bash
  helm uninstall hello-world-demo --namespace hello-world
  ```

  `Install:`
  ```bash
  helm install hello-world-demo hello-world-demo/ --values hello-world-demo/values.yaml --create-namespace --namespace hello-world
  ```