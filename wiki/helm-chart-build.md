## Build Your Own Helm Chart

Using `helm` to build your own chart is super simple! All you need to do is:

```bash
## helm chart <your-chart-name-here>
```

Helm will then populate a directory with the necessary files to deploy your service via a Helm chart. It will look similar to this:

```bash
 .
  ├── Chart.yaml
  ├── charts
  ├── templates
  │   ├── NOTES.txt
  │   ├── _helpers.tpl
  │   ├── deployment.yaml
  │   ├── hpa.yaml
  │   ├── ingress.yaml
  │   ├── service.yaml
  │   ├── serviceaccount.yaml
  │   └── tests
  │       └── test-connection.yaml
  └── values.yaml
```

I made a few adjustments to the base files and will explain in each section:

- Update `values.yaml`:

  - I will be using a custom built docker image with the `latest` tag. Ideally you will update your application and version the `Chart.yaml` with specific application versions that will match your image tag, but for the purpose of the demo I am using a generic chart and the latest tag. 

    ```bash
        image:
          repository: sh4ysum/hello-world
          pullPolicy: IfNotPresent
          tag: "latest"
    ```

  - I need to map a `targetPort` so that we may just serve traffic from the default HTTP port. Update `service.yaml` to include:

    ```bash
    targetPort: {{ .Values.service.targetPort }}
    ```

    - and `values.yaml` to have an additional line in the `service:` block. This is because I will have my NGINX server (within my docker container) listen over port `8080`, but want to map that to port `80` for ease of browsing to the ingress endpoint without the trailing `:8080`.

      ```bash
      targetPort: 8080
      ```

  - I will be using an ingress to route traffic to the service exposed over port `80`.

    ```bash
        ingress:
          enabled: true
          className: "nginx"
          annotations:
              kubernetes.io/ingress.class: nginx
          hosts:
              - host: hello-world-demo.local
              paths:
                  - path: /
                  pathType: ImplementationSpecific
    ```

  - I will be using an HPA for autoscaling based on a CPU metric. 

    **Note:** This will likely not be functional and will report and error because we have not installed the metrics-server. This is expected and will used as a sample alert if you proceed with the [monitoring](../monitoring/README.md) section
   
    ```bash
        autoscaling:
          enabled: true
          minReplicas: 2
          maxReplicas: 10
          targetCPUUtilizationPercentage: 80
    ```

- Add Dependencies in `Chart.yaml`
  ```bash
  ## DEPENDENCIES
  dependencies:
    - name: ingress-nginx
      version: "4.10.0"
      repository: https://kubernetes.github.io/ingress-nginx
  ```

- Subchart values are referenced in the `values.yaml` file as well under the `ingress-nginx` chart name.

  ```bash
  ingress-nginx:
    fullnameOverride: ingress-nginx
    namespaceOverride: ingress
    controller:
      replicaCount: 2
      autoscaling:
        enabled: true
        minReplicas: 2
      minAvailable: 2 ## PODDISRUPTION BUDGET
      service:
        type: LoadBalancer
  ```