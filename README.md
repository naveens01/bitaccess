# Portal Ingress

## Overview

This helm chart will deploy **native Nginx in Spoke environments**. This will expose the loadbalancer via which the communication happens between Hub and Spoke, Customer and Spoke (**Inbound Private Link**). The reason for using native Nginx is due to Ingress controllers' limitation (Combination of Host and Port based routing). This also exposes one loadbalancer as a public LB for inbound traffic to access API products (API Gateway, DevPortal, API Controlplane).

## Kubernetes Resources

Below are the list of Kubernetes resources as part of this helm chart that would get created

| S.No | Kind     | Namespace     | Resource                                   | Conditional Resources                                                                       | Comments                                              |
| ---- | -------- | ------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| 1    | Ingress  |               | ingress                                    | <ol><li>isTlsEnabled</li><li>envRole is "hub" and cloudflare is enabled</li><li>isIngressAnnotationDefined</li><li>isCdmIngressAnnotationDefined</li></ol> |                                                       |
| 2    | Job      | spoke-system  | <ol><li>spoke-custom-domain-nginx-config-job: dpo-cdmncj</li><li>spoke-delete-nginx-config-job: dpo-dncj</li></ol> | <ol><li>envRole is "spoke" and len(customDomain.domains) ne 0</li></ol> |                                                       |
| 3    | ConfigMap | spoke-system  | <ol><li>spoke-nginx-configmap</li></ol>     | <ol><li>envRole is "spoke" and len(customDomain.domains) ne 0</li></ol> | These configmaps have product-specific nginx configurations |

# Secrets Used

| S.No | Name                  | Consumer                                     |
| ---- | --------------------- | -------------------------------------------- |
| 1    | auth-tls-secret       | devportal/cloudflare-ca-cert Ingress controller |
| 2    | Portal-tls-secret     |                                              |
| 3    | Product-secrets       | Jobs: dpo-cdmncj and dpo-dncj                |

# Portal Storage

## Overview

This Helm chart is designed to provision and manage Kubernetes StorageClasses and RBAC resources dynamically based on the provided values. The chart is intended to create storage solutions and access controls tailored to specific cloud providers and environments.

## Details

<ol>
  <li>RBAC configuration to create roles and role bindings for read access to Kubernetes resources.</li>
  <li>readRole value is set to "enabled".</li>
  <li>Rules are set for the roles created and roleBinding is made to bind to the respective role.</li>
  <li>Storage classes are created with parameters different for various use cases such as “slow” or “fast” storage.</li>
</ol>

## Kubernetes Resources

Below are the list of Kubernetes resources as part of this helm chart that would get created

| S.No | Kind            | Namespace  | Resource                   | Conditional Resources                    | Comments                                           |
| ---- | --------------- | ---------- | -------------------------- | ---------------------------------------- | -------------------------------------------------- |
| 1    | RoleBinding     | devportal  | Devportal-read-resource    | Created when readRole is set to “enabled” | Grants permission defined in a role that under roleRef |
| 2    | Role            | devportal  | devportal-read-resource    |                                          | Role definition is designed to grant specific permissions to resources within a namespace, as defined by the RoleBinding |
| 3    | ServiceAccount  | management | aksreaduser                | It is created when the Cloud Provider is not aws | The serviceAccount to which the role is being bound to |
| 4    | Group           |            | eksreaduser                | It is created when the Cloud Provider is aws | The group to which the role is being bound to      |
| 5    | StorageClass    | default    | devportal-block-storage    |                                          | <ul><li>Is configured for slower storage needs, typically using standard disk types</li><li>Is designed for performance-intensive operations, typically using premium disk types</li><li>Is intended for less performance-sensitive workloads, using standard disk</li><li>Is designed for high-performance storage using the Container Storage Interface</li><li>Is intended for workloads with lower performance requirements, leveraging the CSI for standard disk types</li></ul> |

# Portal Tenant Status

## Overview

It is the helm chart that has a cronjob to check tenant status. The job is scheduled that lists takes the list of tenants including the trial and paid tenants, newly created and deleted tenants which is also stored in AWS S3. This chart completely maintains the tenant status with a backup in S3.

## Details

<ol>
  <li>Initialization and cleanup.</li>
  <li>Extracting the environment information.</li>
  <li>Fetching the tenant counts [including TRIAL and PAID tenants].</li>
  <li>Checking files in S3 to copy it locally.</li>
  <li>Calculating Created and Deleted tenants.</li>
  <li>Also calculating the stopped tenant counts [separately for stopped trial tenants in AWS].</li>
  <li>Writing the output to S3 files.</li>
</ol>
