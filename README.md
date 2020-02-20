# aws-helm-service-chart

This chart facilitates the deployment of sample microservice projects on AWS EKS. The service can be any kind of ``https`` based service endpoint 
exposed as docker container.

## Preconditions
1. The service to be installed must be available as docker image.
2. The docker image must be able to be pulled from a valid image repository.
3. This chart has been pushed to a reachable helm chart repository or the whole chart  
   has been integrated into your local project
4. The cluster must be reachable by the Certificate Authority supporting ACME
5. There needs to be a DNS Record Set defined with a wildcard DNS name aliasing the ingress's Load Balancer name.
6. K8S Cert Manager needs to be installed on the cluster
7. You can access your cluster with `kubectl`

## Deployed artifacts
The following components will be deployed with the service.

- Horizontal Pod Autoscaler
- Ingress configuration to an internet facing public ingress controller
- Automated Letsencrypt Issuers for letsencrypt staging and prod environments

## Install the chart
To install an existing service with release name `my-release`, run the following minimum command from the root
folder of your application:

```bash
helm install my-release <HELM_CHART_REPO_REF>\
    --namespace=<NAMESPACE> \
    --set=image.repository=<DOCKER_REPOSITORY_URL> \
    --set=ingress.hosts=<INGRESS_HOST> \
    --set=ingress.class=<INGRESS_CLASS_NAME> \
    --set=containers.readinessProbe.httpGet.path=<READINESS_ENDPOINT_URL>
```

- `NAMESPACE` = Name of an existing namespace where to deploy the service.
- `DOCKER_REPOSITORY_URL` = URL of docker registry to pull image from.
- `INGRESS_HOST` = DNS name of the service under which the ingress should expose this service to the public
- `INGRESS_CLASS_NAME` = Name of the ingress controller to use for this service. 
- `READINESS_ENDPOINT_URL` = The endpoint URL of your service which should be used for the readiness probe, i.e. `/api/v1/status`

This command deploys a default service on an AWS EKS cluster in the provided namespace.

See the [configuration](#Configuration) section for a detailed overview of parameters.

## Uninstall the chart

To uninstall the `my-release` deployment type: `helm delete my-release -n <NAMESPACE>`

## Upload the chart to a chart repository

1. Change the versions in `Chart.yaml` according to the principle of [semantic versioning](https://semver.org/).
2. Next package the chart by typing in `helm package .` in the project's root folder.
3. Push the newly packaged jar to your chart repo: `helm s3 push aws-helm-service-chart-1.2.2.tgz <CHART_REPO>`

## Configuration

The chart can be executed with following parameters:

| Parameter                     | Description   | Example  |
| :---------------------------- |:--------------| :-----   |
| namespace                     | The name of an existing namespace the service should be deployed to. | `default` |
| image.repository              | The name of the AWS ECR image repository to deploy artifacts to. The repository url needs to be provided in the following form: <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/<REPOSITORY_NAME> | `111122223333.dkr.ecr.eu-west-1.amazonaws.com/my-repo` |
| ingress.host                  | A valid DNS name for exposing an ingress route for public access. | `my-service.demo.com` |
| project.includeAwsCredentials | If AWS credentials need to be provided for using other AWS services internally set this flag to `true`. When set to `true` then `secret.aws_accesskey` and `secret.aws_secretkey` need to be provided as well. | `true` if AWS credentials should be included, `false` is the default.|
| secret.aws_accesskey          | It might be necessary to additionally pass the AWS Access Key when using internal AWS services from within your application.  |  AWS Access Key generated for your user  |
| secret.aws_secretkey          | It might be necessary to additionally pass the AWS Secret Key when using internal AWS services from within your application.  |  AWS Secret Key generated for your user  |
| resources.limits.cpu          | Total amount of CPU time that a container can use every 100 ms. See [Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) for a detailed description on resource usage.| `200m` |
| resources.limits.memory       | The memory limit for a Pod. See [Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) for a detailed description on resource usage. | `235M` |
| resources.requests.cpu        | Fractional amount of CPU allowed for a Pod. See [Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) for a detailed description on resource usage.| `150m` |
| resources.requests.memory     | Amount of memory reserved for a Pod. See [Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) for a detailed description on resource usage. | `200M` |
| containers.readinessProbe.httpGet.path  | The endpoint URL of your service which should be used for the readiness probe in the K8S cluster| `/api/v1/status` |
| tls.issuer.spec.acme.email    | A valid e-mail address where letsencrypt may send notifications to regarding certificate requests, renewals, etc.| `your-email@company.com` |
| autoscaling.minReplicas       | Minimum amount of replicas the HPA is allowed for downscaling. | `2` |
| autoscaling.maxReplicas       | Maximum amount of replicas the HPA is allowed for upscaling. | `10` |
| autoscaling.metrics.resource.cpu.targetAverageUtilization | Threshold in percent for CPU usage. Once this value has been reached a new POD will be created. | `80` |

## Testing Horizontal Pod Autoscaling
In order to test the Pod Autoscaler with your deployed service do the following:

1. Download and install a command line performance testing tool, like [ab](https://httpd.apache.org/docs/2.4/programs/ab.html) or [hey](https://github.com/rakyll/hey).  
   The further explanations will use `hey` for testing.
2. Install your application with this chart (see [install the chart](#Install_the_chart))
3. Open a new terminal and execute `kubectl get hpa <SERVICE_NAME> -n <NAMESPACE> -w`
4. Open another terminal and run the load testing tool, i.e. `hey -n 10000 -c 10 https://<SERVICE_URL>`
5. This should cause the HPA to spawn new replicas of your application. Once the load decreases again over a specific  
   period of time the HPA will reduce the amount of replicas automatically.  
   A sample output is shown below:  
   ```
    kubectl get hpa myrelease -n myrelease -w
    NAME                REFERENCE      TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
    myrelease   Deployment/myrelease   2%/80%      2         5         2          5m54s
    NAME                REFERENCE      TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
    myrelease   Deployment/myrelease   1%/80%      2         5         2          7m6s
    myrelease   Deployment/myrelease   130%/80%  2         5         2          9m8s
    myrelease   Deployment/myrelease   130%/80%  2         5         4          9m23s
    myrelease   Deployment/myrelease   115%/80%  2         5         4          10m
    myrelease   Deployment/myrelease   55%/80%    2         5         4          11m
    myrelease   Deployment/myrelease   65%/80%    2         5         4          12m
    myrelease   Deployment/myrelease   67%/80%    2         5         4          13m
    myrelease   Deployment/myrelease   42%/80%    2         5         4          14m
    myrelease   Deployment/myrelease   35%/80%    2         5         4          15m
    myrelease   Deployment/myrelease   30%/80%    2         5         4          16m
    myrelease   Deployment/myrelease   1%/80%      2         5         4          18m
    myrelease   Deployment/myrelease   1%/80%      2         5         3          19m
    myrelease   Deployment/myrelease   1%/80%      2         5         3          20m
    myrelease   Deployment/myrelease   1%/80%      2         5         2          20m
    myrelease   Deployment/myrelease   1%/80%      2         5         2          21m
   ```
7. If you go back to the terminal where you executed the load testing cli command you will see the statistics of  
   of the load test against your service:  
   ```
   Summary:
   Total:        499.2143 secs
   Slowest:      0.6253 secs
   Fastest:      0.0333 secs
   Average:      0.0497 secs
   Requests/sec: 200.3148
  
   Total data:   200000 bytes
   Size/request: 2 bytes

   Response time histogram:
    0.033 [1]     |
    0.093 [88746] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
    0.152 [11002] |■■■■■
    0.211 [177]   |
    0.270 [50]    |
    0.329 [14]    |
    0.389 [4]     |
    0.448 [4]     |
    0.507 [0]     |
    0.566 [1]     |
    0.625 [1]     |

   Latency distribution:
    10% in 0.0365 secs
    25% in 0.0375 secs
    50% in 0.0389 secs
    75% in 0.0431 secs
    90% in 0.0964 secs
    95% in 0.1024 secs
    99% in 0.1300 secs

   Details (average, fastest, slowest):
    DNS+dialup:   0.0000 secs, 0.0333 secs, 0.6253 secs
    DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0125 secs
    req write:    0.0000 secs, 0.0000 secs, 0.0030 secs
    resp wait:    0.0493 secs, 0.0332 secs, 0.6252 secs
    resp read:    0.0001 secs, 0.0000 secs, 0.0072 secs

   Status code distribution:
    [200] 100000 responses
   ```
