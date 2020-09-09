# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.10.2] - 2020-09-09

### Changed

- Fix not working rolling update with helm upgrade. Add timestamp annotation in deployment config
- Update .helmignore to reduce package size

## [1.10.1] - 2020-08-24

### Changed

- Fix missing quotes in Ingress Annotations 

## [1.10.0] - 2020-07-31

### Changed

- Enable [External Secrets](https://github.com/godaddy/kubernetes-external-secrets) to provision secrets from AWS Secrets Manager 
- Add Changelog.md 
- Fix ignored ingress annotations

## [1.9.5] - 2020-06-15

### Changed

- Add possibility to mount yaml config files

## [1.9.4] - 2020-05-29

### Changed

- Add flag to enable ingresses
- Fix errors because of duplicated resources 

## [1.9.3] - 2020-05-28

### Changed

- Add index.yaml to download helm chart directly from GitHub

## [1.9.2] - 2020-05-26

### Changed

- Initial Version


[1.9.2]: https://github.com/DVPE-cloud/aws-helm-service-chart/tree/aws-helm-service-chart-1.9.2
[1.9.3]: https://github.com/DVPE-cloud/aws-helm-service-chart/tree/aws-helm-service-chart-1.9.3
[1.9.4]: https://github.com/DVPE-cloud/aws-helm-service-chart/tree/aws-helm-service-chart-1.9.4
[1.9.5]: https://github.com/DVPE-cloud/aws-helm-service-chart/tree/aws-helm-service-chart-1.9.5
[1.10.0]: https://github.com/DVPE-cloud/aws-helm-service-chart/tree/aws-helm-service-chart-1.10.0
[1.10.1]: https://github.com/DVPE-cloud/aws-helm-service-chart/tree/aws-helm-service-chart-1.10.1
[1.10.2]: https://github.com/DVPE-cloud/aws-helm-service-chart/tree/aws-helm-service-chart-1.10.2

