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
8. [External Secrets](https://github.com/godaddy/kubernetes-external-secrets) needs to be installed on the cluster

## Deployed artifacts
The following components will be deployed with the service.

- Horizontal Pod Autoscaler
- Ingress configuration to an internet facing public Ingress Controller
- Ingress configuration to an intranet facing private Ingress Controller
- Automated Letsencrypt Issuers for letsencrypt staging and prod environments

## Install the chart
To install an existing service with release name `my-release`, run the following minimum command from the root
folder of your application:

### Run without further dependencies

Run the chart if you only want to deploy your service without any further dependencies (this assumes there's no ingress configuration in your environment):

```bash
helm install my-release <HELM_CHART_REPO_REF>\
    --namespace=<NAMESPACE> \
    --set=deployment.spec.image.repository=<DOCKER_REPOSITORY_URL> \
    --set=deployment.spec.serviceAccountName=<SERVICE_ACCOUNT_NAME> \
    --set=deployment.spec.containers.readinessProbe.httpGet.path=<READINESS_ENDPOINT_URL> \
    --set=externalSecret.SERVICE_CREDENTIALS_KEY=<SERVICE_CREDENTIALS_KEY>
```

### Include internal/external Ingress

If you configured ingress controllers in your kubernetes cluster for intranet and extranet access, 
then append the following parameters to enable their usage:

```bash
    --set=ingress.ext.enabled=true \
    --set=ingress.ext.host=<INGRESS_EXT_HOST> \
    --set=ingress.int.enabled=true \
    --set=ingress.int.host=<INGRESS_INT_HOST> \
    --set=tls.cert.int.secret.value=<INGRESS_INT_SECRETS> 
```

Note that the Certificate and Key in the AWS Secret Manager Object (`<INGRESS_INT_SECRETS>`) need to be stored base64 encrypted.

### Include Oauth2 Authentication

If you want to add oauth2 as sidecar in front of your service, append the follow parameters:

```bash
    --set=oauth2.enabled=true \
    --set=oauth2.secret.SIDECAR_CREDENTIALS_KEY=<SIDECAR_CREDENTIALS_KEY> \
    --set=oauth2.config.OIDC_DISCOVERY_URL=<OIDC_DISCOVERY_URL> \
    --set=oauth2.config.OIDC_REDIRECT_URI=<OIDC_REDIRECT_URI> \
    --set=oauth2.config.OIDC_SSL_VERIFY=<OIDC_SSL_VERIFY> \
    --set=oauth2.config.AUTH_TYPE=<AUTH_TYPE> \
    --set=oauth2.config.LOG_LEVEL=<LOG_LEVEL> \
    --set=oauth2.sidecar.image.repository=<SIDECAR_REPOSITORY> \
    --set=oauth2.sidecar.image.name=<SIDECAR_IMAGE} \
    --set=oauth2.sidecar.image.tag=<SIDECAR_TAG>
```

Note that `oauth2.secret.*` values will be base64 encoded while creating the Kubernetes Secrets, so there is no need to pre-encrypt them

### Include Redis Session-Cache for Oauth2

If you want to use redis as session-cache for the auth-sidecar container, append the follow parameters
You also need to configure and enable oauth2 parameters (see Include Oauth2 Authentication).

```bash
    --set=oauth2.cache.SESSION_STORAGE=<SESSION_STORAGE> \
    --set=oauth2.cache.SESSION_STORAGE_HOST=<SESSION_STORAGE_HOST> \
    --set=oauth2.cache.SESSION_STORAGE_PORT=<SESSION_STORAGE_PORT> \
```

### Include DataDog Monitoring

Finally if you like to add DataDog Monitoring for the included service append the following parameters:

```bash
    --set=datadog.enabled=true
    --set=datadog.source.service="java"
```

### Variable Definitions

- `NAMESPACE` = Name of an existing namespace where to deploy the service.
- `SERVICE_ACCOUNT_NAME` = Name of the K8S Service Account a POD should use
- `DOCKER_REPOSITORY_URL` = URL of docker registry to pull image from.
- `INGRESS_EXT_HOST` = DNS name of the service under which the external ingress should expose this service to the public
- `INGRESS_INT_HOST` = DNS name of the service under which the internal ingress should expose this service to the corporate network
- `INGRESS_INT_SECRETS` = Reference to AWS Secret Manager which include Certificate (`tls.crt`) and Key (`tls.key`) for TLS configuration of internal Ingress Controller
- `READINESS_ENDPOINT_URL` = The endpoint URL of your service which should be used for the readiness probe, i.e. `/api/v1/status`
- `SERVICE_CREDENTIALS_KEY` = Reference to AWS Secret Manager with all sensitive data for the service(e.g. database password)
- `SIDECAR_CREDENTIALS_KEY` = Reference to AWS Secret Manager with all sensitive data for the sidecar (e.g. Client ID, Client Secret, Cache Password, Session Secret)
- `OIDC_DISCOVERY_URL` = URL from IDP Authentication
- `OIDC_REDIRECT_URI` = Redirect URL
- `OIDC_SSL_VERIFY`= Enable or disable SSL Certificate Verification
- `AUTH_TYPE` = Authentication Mode (UI or BACKEND)
- `LOG_LEVEL` = Log Level (Default: info)
- `SIDECAR_REPOSITORY` = Repostory of Oauth2 Container Image
- `SIDECAR_IMAGE` = Name of Oauth2 Container Image
- `SIDECAR_TAG` = Container Image Tag
- `SESSION_STORAGE` = Define the cache storage
- `SESSION_STORAGE_HOST` = Cache System Hostname/IP Address
- `SESSION_STORAGE_PORT` = Cache System Port Number
- `SESSION_STORAGE_CACHE_TTL` = Redis Key Expire TTL


See the [configuration](#Configuration) section for a detailed overview of parameters.

### Define additional parameters for deployed services

Sometimes the deployed service needs some extra parameters or secrets to run. In such cases an additional ConfigMap or Secret will be created to pass the necessary values to the container as environmental variables. 

If you want to pass a YAML configuration file, set `yamlConfigFileApplied` to `true` and pass it under `yamlConfigFile`. Your file will be mounted as a `ConfigMap` volume under `/etc/config/config.yaml` and you can feed it into your application.

To do that you can include a supplementary values.yaml file in your own project and format it like in the code snippet below:

```
additionalparameters:
  configMapApplied: true # flag to tell a configmap is defined
  secretsMapApplied: true # flag to tell secrets are defined
  yamlConfigFileApplied: true # flag to tell yaml config file is defined
  config:
    <PARAM_NAME>: <PARAM_VALUE>
    <PARAM_NAME2>: <PARAM_VALUE2>
    ...
  secret:
    <SECRET_NAME>: <SECRET_VALUE>
    ...
  yamlConfigFile:
    <ROOT_YAML_PROP1>: 
        ...
    <ROOT_YAML_PROP2>: 
        ...
```

You can then call the chart by passing the additional values.yaml as command line argument:
```
helm install <SERVICE_NAME> -f values.yaml
```

All secret values defined in this file will be base64 encoded while creating the Kubernetes Secrets, so there is no need to pre-encrypt them.

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
| deployment.spec.serviceAccountName | The K8S Service Account a Pod should use. | `default` |
| deployment.spec.image.repository | The name of the AWS ECR image repository to deploy artifacts to. The repository url needs to be provided in the following form: <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/<REPOSITORY_NAME> | `111122223333.dkr.ecr.eu-west-1.amazonaws.com/my-repo` |
| ingress.ext.enabled           | Set to `true` if a public (internet facing) ingress controller has been configured in your cluster. Default is set to `false`| `false` |
| ingress.ext.host              | A valid DNS name for exposing an ingress route for public access. `ingress.ext.enabled` must be set to `true`| `my-service.<AWS_REGION>.bmw.cloud` |
| ingress.int.enabled           | Set to `true` if a private (intranet facing) ingress controller has been configured in your cluster. Default is set to `false` | `false` |
| ingress.int.host              | A valid DNS name for exposing an ingress route for private access. `ingress.int.enabled` must be set to `true`| `my-service.<AWS_REGION>.cloud.bmw` |
| tls.cert.int.secret.value     | Reference to AWS Secret Manager which includes a `tls.crt` as cert and a `tls.key` as key. `NOTE`: This is only relevant if your K8S dev namespace is labeled `private`. It must be base64 pre-encrypted. | `tls.cert.int.secret` |
| project.includeAwsCredentials | If AWS credentials need to be provided for using other AWS services internally set this flag to `true`. When set to `true` then `secret.aws_accesskey` and `secret.aws_secretkey` need to be provided as well. | `true` if AWS credentials should be included, `false` is the default.|
| secret.aws_accesskey          | It might be necessary to additionally pass the AWS Access Key when using internal AWS services from within your application. Must be base64 pre-encrypted |  AWS Access Key generated for your user  |
| secret.aws_secretkey          | It might be necessary to additionally pass the AWS Secret Key when using internal AWS services from within your application. Must be base64 pre-encrypted |  AWS Secret Key generated for your user  |
| externalSecret.SERVICE_CREDENTIALS_KEY | All sensitive data needed by your application should be stored in a AWS Secret Manager Object. Each key should be named like your needed environment variable | `service.permission-checker` |
| deployment.spec.resources.limits.cpu | Total amount of CPU time that a container can use every 100 ms. See [Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) for a detailed description on resource usage.| `200m` |
| deployment.spec.resources.limits.memory | The memory limit for a Pod. See [Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) for a detailed description on resource usage. | `235M` |
| deployment.spec.resources.requests.cpu | Fractional amount of CPU allowed for a Pod. See [Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) for a detailed description on resource usage.| `150m` |
| deployment.spec.resources.requests.memory | Amount of memory reserved for a Pod. See [Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/) for a detailed description on resource usage. | `200M` |
| deployment.spec.containers.readinessProbe.httpGet.path | The endpoint URL of your service which should be used for the readiness probe in the K8S cluster| `/api/v1/status` |
| tls.issuer.spec.acme.email    | A valid e-mail address where letsencrypt may send notifications to regarding certificate requests, renewals, etc.| `your-email@company.com` |
| autoscaling.minReplicas       | Minimum amount of replicas the HPA is allowed for downscaling. | `2` |
| autoscaling.maxReplicas       | Maximum amount of replicas the HPA is allowed for upscaling. | `10` |
| autoscaling.metrics.resource.cpu.targetAverageUtilization | Threshold in percent for CPU usage. Once this value has been reached a new POD will be created. | `80` |
| oauth2.enabled | Enable or disable Oauth2 Authentification Sidecar | `true` |
| oauth2.sidecar.image.repository | Auth Sidecar Container Image URL | Example (not exist): `hub.docker.com/sidecar` |
| oauth2.sidecar.image.name | Auth Sidecar Container Image Name | `auth-sidecar` |
| oauth2.sidecar.image.tag | Auth Sidecar Container Image Tag | `v1.1.0` |
| oauth2.sidecar.image.pullPolicy | Kubernetes Image PullPolicy | `Always` |
| oauth2.sidecar.servicePort | On which port the service runs | `8443` |
| oauth2.secret.SIDECAR_CREDENTIALS_KEY | Reference to a AWS Secret Manager Object with all relevant sidecar credentials. Should contain at least ClientId, Client Secret, Session Secret and Cache Auth Password | Example: `service.permission-checker.auth-sidecar` |
| oauth2.config.OIDC_DISCOVERY_URL | URL from IDP Discovery Service | `https://idp.domain/auth/discovery` |
| oauth2.config.OIDC_REDIRECT_URI | Callback URL | `https://127.0.0.1:8443/callback` |
| oauth2.config.OIDC_SCOPE | Oauth2 Scope | `openid` |
| oauth2.config.OIDC_TOKEN_ENDPOINT_AUTH_METHOD | Oauth2 token endpoint authentication method | `client_secret_basic` |
| oauth2.config.OIDC_SSL_VERIFY | Enable or disable ssl certificat validation | `yes` |
| oauth2.config.OIDC_TOKEN_EXPIRY_TIME | TTL of accesstoken in seconds | `7200` |
| oauth2.config.OIDC_RENEW_ACCESS_TOKEN | Whether to renew the access token once it has expired | `true` |
| oauth2.config.TARGET_HOST | URL of the actual service behind this auth sidcar within the same pod | `http://127.0.0.1` |
| oauth2.config.TARGET_PORT | Port of the actual service behind this auth sidcar within the same pod | `8080` |
| oauth2.config.LOG_LEVEL | Log level of the auth-sidecar reverse Proxy | `info` |
| oauth2.config.AUTH_TYPE | Client Authentication Type (UI/BACKEND) | `UI` |
| oauth2.cache.SESSION_STORAGE | Name of the storage system, currently only cookie and redis are available, default is cookie | `redis` |
| oauth2.cache.SESSION_STORAGE_HOST | Hostname (FQDN) or IP Address of the session cache server/cluster | `127.0.0.1` |
| oauth2.cache.SESSION_STORAGE_PORT | Port Number | `6379` |
| oauth2.cache.SESSION_STORAGE_CACHE_TTL | Redis Expire TTL Value in seconds (default 2h) | `7200` |
| datadog.enabled | `true` if DataDog annotations should be used, `false` otherwise | `false` |
| datadog.source.service| The name of the DataDog source the service should be instrumented with. | `java` |

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
