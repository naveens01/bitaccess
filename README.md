# Portal Ingress

## Overview

This helm chart will deploy resources necessary for managing ingress and custom domain configurations for the Portal application.

## Kubernetes Resources

Below are the list of Kubernetes resources as part of this helm chart that would get created:

| S.No | Kind   | Namespace   | Resource                  | Conditional Resources                     | Comments                                      |
| ---- | ------ | ----------- | ------------------------- | ----------------------------------------- | --------------------------------------------- |
| 1    | Ingress| <namespace> | <ingress-resource>        | isTlsEnabled                              |                                               |
|      |        |             |                           | envRole is "hub" and                      |                                               |
|      |        |             |                           | cloudflare is enabled                     |                                               |
|      |        |             |                           | isIngressAnnotationDefined                |                                               |
|      |        |             |                           | isCdmIngressAnnotationDefined             |                                               |
| 2    | Job    | spoke-system| <job-name>                | envRole is "spoke" and                    |                                               |
|      |        |             | spoke-custom-domain-nginx | len(customDomain.domains) > 0             |                                               |
|      |        |             | -config-job               |                                           |                                               |
|      |        |             | dpo-cdmncj                |                                           |                                               |
|      |        |             | spoke-delete-nginx-config | envRole is "spoke" and                    |                                               |
|      |        |             | -job                      | len(customDomain.domains) > 0             |                                               |
| 3    | ConfigMap | spoke-system | spoke-nginx-configmap | envRole is "spoke" and                    | These configmaps have product specific nginx |
|      |        |             |                           | len(customDomain.domains) > 0             | configurations                                |

# Portal Storage

## Overview

This section covers the storage-related Kubernetes resources that will be deployed as part of this helm chart.

| S.No | Kind           | Namespace | Resource                 | Conditional Resources                | Comments                       |
| ---- | -------------- | --------- | ------------------------ | ------------------------------------ | ------------------------------ |
| 1    | RoleBinding    | devportal | devportal-read-resource  | readRole is enabled                  |                                |
| 2    | Role           | devportal | devportal-read-resource  | cloudProvider is aws                 |                                |
| 3    | ServiceAccount | management| aksreaduser              | cloudProvider not aws                |                                |
|      |                |           |                          | readRole is enabled                  |                                |
| 4    | Group          | <namespace> | eksreaduser            | cloudProvider not aws                |                                |
|      |                |           |                          | readRole is enabled                  |                                |
| 5    | StorageClass   | <namespace> | devportal-block-storage |                                      |                                |
