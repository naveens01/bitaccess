# Helm Charts for DevPortal

## Portal Ingress

### Overview

This Helm chart will deploy **native Nginx in Spoke environments**. This will expose the load balancer via which the communication happens between Hub and Spoke, Customer and Spoke (**Inbound Private Link**). The reason for using native Nginx is due to Ingress controllers' limitation (Combination of Host and Port based routing). This also exposes one load balancer as a public LB for inbound traffic to access API products (API Gateway, DevPortal, API Controlplane).

### Kubernetes Resources

Below are the list of Kubernetes resources as part of this Helm chart that would get created:

| S.No | Kind      | Namespace    | Resource                                    | Conditional Resources                                                                      | Comments                                              |
| ---- | --------- | ------------ | ------------------------------------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------- |
| 1    | Ingress   |              | ingress                                     | <ol><li>isTlsEnabled</li><li>envRole is "hub" and cloudflare is enabled</li><li>isIngressAnnotationDefined</li><li>isCdmIngressAnnotationDefined</li></ol> |                                                       |
| 2    | Job       | spoke-system | <ol><li>spoke-custom-domain-nginx-config-job: dpo-cdmncj</li><li>spoke-delete-nginx-config-job: dpo-dncj</li></ol> | <ol><li>envRole is "spoke" and len(customDomain.domains) ne 0</li></ol>                    |                                                       |
| 3    | ConfigMap | spoke-system | <ol><li>spoke-nginx-configmap</li></ol>     | <ol><li>envRole is "spoke" and len(customDomain.domains) ne 0</li></ol>                    | These configmaps have product-specific Nginx configurations |

### Secrets Used

| S.No | Name               | Consumer                                     |
| ---- | ------------------ | -------------------------------------------- |
| 1    | auth-tls-secret    | devportal/cloudflare-ca-cert Ingress controller |
| 2    | Portal-tls-secret  |                                              |
| 3    | Product-secrets    | Jobs: dpo-cdmncj and dpo-dncj                |

## Portal Storage

### Overview

This Helm chart is designed to provision and manage Kubernetes StorageClasses and RBAC resources dynamically based on the provided values. The chart is intended to create storage solutions and access controls tailored to specific cloud providers and environments.

### Details

<ol>
  <li>RBAC configuration to create roles and role bindings for read access to Kubernetes resources.</li>
  <li>readRole value is set to "enabled".</li>
  <li>Rules are set for the roles created and roleBinding is made to bind to the respective role.</li>
  <li>Storage classes are created with parameters different for various use cases such as “slow” or “fast” storage.</li>
</ol>

### Kubernetes Resources

Below are the list of Kubernetes resources as part of this Helm chart that would get created:

| S.No | Kind            | Namespace  | Resource                   | Conditional Resources                    | Comments                                           |
| ---- | --------------- | ---------- | -------------------------- | ---------------------------------------- | -------------------------------------------------- |
| 1    | RoleBinding     | devportal  | Devportal-read-resource    | Created when readRole is set to “enabled” | Grants permission defined in a role that under roleRef |
| 2    | Role            | devportal  | devportal-read-resource    |                                          | Role definition is designed to grant specific permissions to resources within a namespace, as defined by the RoleBinding |
| 3    | ServiceAccount  | management | aksreaduser                | It is created when the Cloud Provider is not aws | The serviceAccount to which the role is being bound to |
| 4    | Group           |            | eksreaduser                | It is created when the Cloud Provider is aws | The group to which the role is being bound to      |
| 5    | StorageClass    | default    | devportal-block-storage    |                                          | <ul><li>Is configured for slower storage needs, typically using standard disk types</li><li>Is designed for performance-intensive operations, typically using premium disk types</li><li>Is intended for less performance-sensitive workloads, using standard disk</li><li>Is designed for high-performance storage using the Container Storage Interface</li><li>Is intended for workloads with lower performance requirements, leveraging the CSI for standard disk types</li></ul> |

## Portal Tenant Status

### Overview

It is the Helm chart that has a cronjob to check tenant status. The job is scheduled to list tenants including trial and paid tenants, newly created and deleted tenants which are also stored in AWS S3. This chart completely maintains the tenant status with a backup in S3.

### Details

<ol>
  <li>Initialization and cleanup.</li>
  <li>Extracting the environment information.</li>
  <li>Fetching the tenant counts [including TRIAL and PAID tenants].</li>
  <li>Checking files in S3 to copy them locally.</li>
  <li>Calculating Created and Deleted tenants.</li>
  <li>Also calculating the stopped tenant counts [separately for stopped trial tenants in AWS].</li>
  <li>Writing the output to S3 files.</li>
</ol>

### Kubernetes Resources

Below are the list of Kubernetes resources as part of this Helm chart that would get created:

| S.No | Kind      | Namespace  | Resource                            | Conditional Resources | Comments                                      |
| ---- | --------- | ---------- | ----------------------------------- | --------------------- | --------------------------------------------- |
| 1    | CronJob   | devportal  | portalbundle-generate-tenants-status-job |                       | Check the status of the tenants and store in S3 |
| 2    | ConfigMap | devportal  | generate-tenants-status-conf        |                       | Used by portalbundle-generate-tenants-status-job |

### Secrets Used

| S.No | Secret           | Consumer                                      |
| ---- | ---------------- | --------------------------------------------- |
| 1    | product-secrets  | CronJob - portalbundle-generate-tenants-status-job |

## DevPortal

### Overview

### Kubernetes Resources

Below are the list of Kubernetes resources as part of this Helm chart that would get created:

| S.No | Kind         | Namespace  | Resource                                     | Conditional Resources                                                                                      | Comments                                                                                                                                                           |
| ---- | ------------ | ---------- | -------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1    | CronJob      |            | <ol><li>dpo-dtj</li></ol>                    | <ol><li>Created if tier is not paid and values.globalReplicas is not 0</li></ol>                             | Stopping the tenant based on the days difference between the current date and created date                                                                         |
| 2    | Job          | devportal  | <ol><li>es-delesi</li><li>es-mubj</li><li>migration-bkpjob</li><li>migration-restore</li><li>dpo-final-backup</li><li>dpo-restore-snapshot</li></ol> | <ol><li>Created if tier is paid and retainTrialIndicesStatus is set to "disabled"</li><li>Created if tier is paid and enableReadOnly is set to "true"</li><li>Created if tier is paid and isManualBackup is set as "enabled"</li><li>Created if tier is paid and isManualRestore is set as “enabled”</li><li>Created if finalSnapshot is set to "enabled"</li><li>Created if restoreLatestSnapshot is "enabled"</li></ol> | <ol><li>Delete the Elasticsearch indices if it reaches the max-time</li><li>Create Dev portal’s major upgrade Elasticsearch snapshot for tenants</li><li>Elasticsearch backup repo is created in which snapshot is created. Tenant chart values are updated finally</li><li>An es restore repo is created and the snapshots are restored. Finally, update the tenant chart values</li><li>Creation of final es snapshot for the tenants and deletion of orphan jobs</li><li>Register es repo, restore the snapshots, update the registered repo. Restarting pods and deleting jobs are executed</li></ol> |
| 3    | ConfigMap    |            | <ol><li>portalbundle-ctp-config</li><li>portalbundle-spring-config</li><li>deallocate-tenant-conf</li><li>final-backup-conf</li><li>elasticsearch-backup-config</li><li>es-major-upgrade-snapshot-config</li><li>migration-bkpconfig</li><li>migration-restore-config</li></ol> | <ol><li>If tier is not “paid”</li><li>FinalSnapshot is set to “enabled”</li><li>If tier is “paid”</li><li>If tier is “paid” and enableReadOnly is set to "true"</li><li>If tier is “paid” and isManualBackup is set to "enabled"</li><li>If tier is “paid” and isManualBackup is set to "enabled"</li></ol> | Configuring the DevPortal application bundle and its dynamic generation based on environment and tenant-specific details.                                            |
| 4    | Deployment   |            | <ol><li>portalbundle-deploy</li><li>esexp-deploy</li></ol>                   | <ol><li>If the installation type is "bundle" and either the tier is not "paid" with global replicas not equal to
