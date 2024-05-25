# Portal Ingress

## Overview

This helm chart will deploy **native Nginx in Spoke environments**. This will expose the loadbalancer via which the communication happens between Hub and Spoke, Customer and Spoke (**Inbound Private Link**). The reason for using native Nginx is due to Ingress controllers' limitation (Combination of Host and Port based routing). This also exposes one loadbalancer as a public LB for inbound traffic to access API products (API Gateway, DevPortal, API Controlplane).

## Kubernetes Resources

Below are the list of Kubernetes resources as part of this helm chart that would get created

| S.No | Kind   | Namespace | Resource | Conditional Resource | Comments  |
| ---- | ------ | --------- | -------- | -------------------- | --------- |
| 1    | Ingress |           | ingress | <ol><li>isTlsEnabled</li><li>envRole is "hub" and cloudflare is enabled</li><li>isIngressAnnotationDefined</li><li>isCdmIngressAnnotationDefined</li></ol> |           |
| 2    | Job    | spoke-system | <ol><li>spoke-custom-domain-nginx-config-job: dpo-cdmncj</li><li>spoke-delete-nginx-config-job: dpo-dncj</li></ol> | <ol><li>envRole is "spoke" and len(customDomain.domains) ne 0</li></ol> |           |
| 3    | ConfigMap | spoke-system | <ol><li>spoke-nginx-configmap</li></ol> | <ol><li>envRole is "spoke" and len(customDomain.domains) ne 0</li></ol> | These configmaps have product-specific nginx configurations |

# Portal Storage

## Overview

Below are the list of Kubernetes resources as part of this helm chart that would get created

| S.No | Kind          | Namespace  | Resource                  | Conditional Resource     | Comments |
| ---- | ------------- | ---------- | ------------------------- | ------------------------ | -------- |
| 1    | RoleBinding   | devportal  | Devportal-read-resource   | <ol><li>readRole is enabled</li></ol> | |
| 2    | Role          | devportal  | devportal-read-resource   | <ol><li>cloudProvider is aws</li></ol> | |
| 3    | ServiceAccount | management | aksreaduser               | <ol><li>Cloudprovider not aws</li><li>readRole is enabled</li></ol> | |
| 4    | Group         |            | eksreaduser               | <ol><li>Cloudprovider not aws</li><li>readRole is enabled</li></ol> | |
| 5    | StorageClass  |            | devportal-block-storage   | | |
