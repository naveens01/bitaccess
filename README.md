# Helm Charts for DevPortal

## Portal Ingress

### Overview

This Helm chart will deploy **native Nginx in Spoke environments**. This will expose the load balancer via which the communication happens between Hub and Spoke, Customer and Spoke (**Inbound Private Link**). The reason for using native Nginx is due to Ingress controllers' limitation (Combination of Host and Port based routing). This also exposes one load balancer as a public LB for inbound traffic to access API products (API Gateway, DevPortal, API Controlplane).

### Kubernetes Resources

Below are the list of Kubernetes resources as part of this Helm chart that would get created:

| S.No | Kind           | Namespace    | Resource                 | Conditional Resources                                | Comments                                    |
| ---- | -------------- | ------------ | ------------------------ | ---------------------------------------------------- | ------------------------------------------- |
| 1    | Ingress        |              | Portal-ing-http          |                                                      | Handles domain-specific routing with custom annotations and TLS configurations |
| 2    | Job            | Spoke-system | dpo-cdmncj               | <ol><li>envRole is "spoke" and len(customDomain.domains) ≠ 0</li></ol> | Job that updates and cleans up Nginx configuration based on custom domains |
| 3    | ConfigMap      | Spoke-system | dpo-nc                   | <ol><li>envRole is "spoke" and len(customDomain.domains) ≠ 0</li></ol> | Product-specific Nginx configurations        |
| 4    | ServiceAccount | Spoke-system | apigwnginxuser           |                                                      | Used by the dpo-cdmncj job                   |

### Secrets Used

| S.No | Name                | Consumer                                     |
| ---- | ------------------- | -------------------------------------------- |
| 1    | auth-tls-secret     | devportal/cloudflare-ca-cert Ingress controller |
| 2    | Portal-tls-secret   |                                              |
| 3    | Product-secrets     | Jobs: dpo-cdmncj and dpo-dncj                |

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

| S.No | Kind           | Namespace  | Resource                 | Conditional Resources                                    | Comments                                                                                       |
| ---- | -------------- | ---------- | ------------------------ | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------|
| 1    | RoleBinding    | devportal  | Devportal-read-resource  | <ol><li>readRole is set to "enabled"</li></ol>           | Grants permissions defined in roles under roleRef                                             |
| 2    | Role           | devportal  | devportal-read-resource  |                                                          | Grants specific permissions to resources within a namespace                                    |
| 3    | ServiceAccount | management | aksreaduser              | <ol><li>Cloud Provider is not AWS</li></ol>              | The serviceAccount to which the role is being bound                                            |
| 4    | Group          |            | eksreaduser              | <ol><li>Cloud Provider is AWS</li></ol>                  | The group to which the role is being bound                                                     |
| 5    | StorageClass   | default    | devportal-block-storage  |                                                          | Configured for slower storage needs, typically using standard disk types                       |
| 6    | StorageClass   |            | devportal-block-storage-fast |                                                      | Designed for performance-intensive operations, typically using premium disk types              |
| 7    | StorageClass   |            | devportal-block-storage-slow |                                                      | Intended for less performance-sensitive workloads, using standard disk                         |
| 8    | StorageClass   |            | devportal-block-storage-fast-csi |                                                | Designed for high-performance storage using the Container Storage Interface                    |
| 9    | StorageClass   |            | devportal-block-storage-slow-csi |                                                | Intended for workloads with lower performance requirements, leveraging the CSI for standard disk types |

## Portal Tenant Status

### Overview

This Helm chart includes a cron job to check tenant status. The job is scheduled to list tenants, including trial and paid tenants, newly created and deleted tenants, and store the information in AWS S3. It maintains tenant status with a backup in S3.

### Details

<ol>
  <li>Initialization and cleanup.</li>
  <li>Extracting environment information to fetch files from S3.</li>
  <li>Fetching trial and paid tenant counts.</li>
  <li>Calculating newly created and deleted tenants by comparing files.</li>
  <li>Fetching stopped tenants and storing the final output in AWS S3.</li>
</ol>

### Kubernetes Resources

Below are the list of Kubernetes resources as part of this Helm chart that would get created:

| S.No | Kind      | Namespace | Resource                             | Conditional Resources | Comments                                          |
| ---- | --------- | --------- | ------------------------------------ | --------------------- | ------------------------------------------------ |
| 1    | CronJob   | devportal | portalbundle-generate-tenants-status-job |                       | Check the status of the tenants and stores that in S3 |
| 2    | ConfigMap | devportal | generate-tenants-status-conf         |                       | Used by portalbundle-generate-tenants-status-job |

### Secrets Used

| S.No | Secret         | Consumer                                    |
| ---- | -------------- | ------------------------------------------- |
| 1    | product-secrets| CronJob - portalbundle-generate-tenants-status-job |

## DevPortal

### Kubernetes Resources

Below are the list of Kubernetes resources as part of this Helm chart that would get created:

| S.No | Kind              | Namespace | Resource                                    | Conditional Resources                                      | Comments                                                                                                     |
| ---- | ----------------- | --------- | ------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| 1    | CronJob           |           | dpo-dtj                                     | <ol><li>tier is not paid</li><li>values.globalReplicas ≠ 0</li></ol> | Stopping the tenant based on the days difference between the current date and created date                   |
| 2    | Job               | devportal | <ol><li>es-delesi</li><li>es-mubj</li><li>migration-bkpjob</li><li>migration-restore</li><li>dpo-final-backup</li><li>dpo-restore-snapshot</li></ol> | Various conditions                              | Various jobs related to Elasticsearch snapshot management, tenant chart updates, and backups                 |
| 3    | ConfigMap         |           | <ol><li>portalbundle-ctp-config</li><li>portalbundle-spring-config</li><li>deallocate-tenant-conf</li><li>final-backup-conf</li><li>elasticsearch-backup-config</li><li>es-major-upgrade-snapshot-config</li><li>migration-bkpconfig</li><li>migration-restore-config</li></ol> | Various conditions                              | Configuring the DevPortal application bundle and its dynamic generation based on environment and tenant-specific details |
| 4    | Deployment        |           | <ol><li>Portalbundlt-deploy</li><li>esexp-deploy</li></ol> | Various conditions                                          | Deployment configurations based on installation type and tier                                               |
| 5    | Service           |           | <ol><li>portalbundle-svc-http</li><li>portalbundle-svc-headless</li><li>esexp-svc-http</li><li>elasticsearch-svc-http</li><li>elasticsearch-svc-headless</li></ol> | Various conditions                                          | Service configurations for portalbundle and Elasticsearch                                                     |
| 6    | PodDisruptionBudget |         | <ol><li>portalbundle-pdb</li><li>elasticsearch-pdb</li></ol> | Various conditions                                          | Ensuring high availability and stability during disruptions                                                  |
| 7    | StatefulSet       |           | <ol><li>portalbundle-sts</li><li>elasticsearch-sts</li></ol> | Various conditions                                          | StatefulSet configurations for persistent applications and data                                              |
| 8    | RoleBinding       | devportal | Devportal-read-resource                     | <ol><li>readRole is set to "enabled"</li></ol>              | Grants permissions defined in roles under roleRef                                                             |
| 9    | Role              | devportal | devportal-read-resource                     |                                                            | Grants specific permissions to resources within a namespace                                                  |
| 10   | Ingress           |           | <ol><li>Portal-ing-http</li><li>Portal-ing-custom</li></ol> |                                                            | Handles domain-specific routing with custom annotations and TLS configurations                                |

## Secrets Used

| S.No | Name                         | Consumer                                                   |
| ---- | ---------------------------- | ---------------------------------------------------------- |
| 1    | Portal-logs-backup-secret    | dpo-dtj, migration-bkpjob, migration-restore, dpo-final-backup |
| 2    | Product-secrets              | es-mubj, es-delesi                                          |
| 3    | portal-app-secret            | portalbundle-deploy, portalbundle-sts                       |
| 4    | elasticsearch-secret         | elasticsearch-sts                                           |

## Monitoring

### Overview

The Monitoring Chart is designed to streamline and enhance the observability and reliability of our infrastructure. It includes components to manage and monitor the system and automates the process of deleting orphan storage, manages Elasticsearch (ES) backup, and integrates with monitoring tools like Prometheus via the serviceMonitors.

### Details

- **Orphan Storage Deletion**: The cronjob runs the script stored in the ConfigMap at scheduled intervals to delete orphaned storage data.
- **ES Backup Status**: Another cronjob periodically checks the status of Elasticsearch backups and logs the results.
- **ServiceMonitor**: Prometheus scrapes metrics from the services specified in the ServiceMonitor configuration, providing real-time monitoring.
- **ServiceAccount**: Ensures secure and managed access to Kubernetes resources for the scripts and jobs.

### Kubernetes Resources

Below are the list of Kubernetes resources as part of this helm chart that would get created:

| S.No | Kind            | Namespace | Resource                       | Conditional Resources                                         | Comments                                                                                           |
| ---- | --------------- | --------- | ------------------------------ | ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| 1    | CronJob         | devportal | delete-orphan-snapshot-folders |                                                               |                                                                                                    |
|      |                 |           | push-es-backup-status          | <ol><li>Created when secondaryDeploymentStatus is enabled or dcType is primary</li></ol> | <ol><li>Managing and cleaning up the tenant related data stored in AWS S3 or Azure blob storage</li><li>PATCH call is made that sends the payload that contains backup status</li></ol> |
| 2    | ServiceAccount  | devportal | Devportal-sa                   | Created if cloudProvider is AWS                                | Used by delete-orphan-snapshot-folder cronjob                                                      |
| 3    | Config          | devportal | delete-orphan-storage-conf     |                                                               |                                                                                                    |
|      |                 |           | push-backup-status-conf        | <ol><li>Created when secondaryDeploymentStatus is enabled or dcType is primary</li></ol> | Used by delete-orphan-snapshot-folders and push-es-backup-status cronjobs                          |
| 4    | ServiceMonitor  |           | devportal-es-exporter          |                                                               | The data is being scraped by kube-prometheus-agent                                                 |

## External Secrets

| S.No | Name                          | Consumer                                                       | Key                                         |
| ---- | ----------------------------- | -------------------------------------------------------------- | ------------------------------------------- |
| 1    | dpo-tms-secret                | delete-orphan-snapshot-folders cronjob, push-es-backup-status cronjob | OPSGENIE_AUTH_KEY – DPO Opsgenie Auth key   |
| 2    | Product-secrets [hub]         | delete-orphan-snapshot-folders cronjob, push-es-backup-status cronjob | <ol><li>KC_TMS_POWER_CLIENT_ID - Dpo TMS's Power user Keycloak Client ID</li><li>KC_TMS_POWER_CLIENT_SECRET - Dpo TMS's Power user Keycloak Client secret</li></ol> |
| 3    | product-secrets-spoke         | delete-orphan-snapshot-folders cronjob, push-es-backup-status cronjob | <ol><li>KC_TMS_POWER_CLIENT_ID - Dpo TMS's Power user Keycloak Client ID</li><li>KC_TMS_POWER_CLIENT_SECRET - Dpo TMS's Power user Keycloak Client secret</li></ol> |

## Azure

| S.No | Name                          | Consumer                                                       | Key                                                         |
| ---- | ----------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------- |
| 1    | dpo-tms-secret                | delete-orphan-snapshot-folders cronjob, push-es-backup-status cronjob | AZURE_STORAGE_CONTAINER_NAME – Container name of Dpo azure blob storage |
| 2    | product-secrets               | delete-orphan-snapshot-folders cronjob, push-es-backup-status cronjob | <ol><li>AZURE_STORAGE_ACCOUNT – Dpo Azure blob storage account</li><li>AZURE_STORAGE_KEY - Dpo Azure blob storage key</li></ol> |

## Devportal-110

### Kubernetes Resources

Below are the list of Kubernetes resources as part of this helm chart that would get created:

| S.No | Kind              | Namespace | Resource                          | Conditional Resources                                          | Comments                                                                 |
| ---- | ----------------- | --------- | --------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------ |
| 1    | CronJob           |           | elasticsearch-sj                  |                                                                |                                                                          |
|      |                   |           |                                   | <ol><li>es manage snapshot</li></ol>                            |                                                                          |
|      | Job               |           | dr-restore                        |                                                                |                                                                          |
|      |                   |           | elasticsearch-create-repo         | <ol><li>If manualRestoreScenario is "dr"</li><li>If Release.IsUpgrade is false</li></ol> |                                                                          |
| 3    | ConfigMap         |           | <ol><li>portalbundle-ctp-config</li><li>portalbundle-spring-config</li><li>final-backup-conf</li><li>dr-restore-config</li><li>elasticsearch-backup-config</li></ol> | <ol><li>If finalSnapshot is enabled</li><li>If manualRestoreScenario is set as "dr"</li><li>If tier is "paid"</li></ol> |                                                                          |
| 4    | NetworkPolicy     |           | <ol><li>deny-ingress-netpol</li><li>portalbundle-netpol</li><li>elasticsearch-netpol</li><li>elasticsearch-exporter-netpol</li><li>jobs-netpol</li></ol> | <ol><li>networkPolicy.status is enabled and networkPolicy.type is kubernetes</li></ol> |                                                                          |
| 5    | Ingress           |           | <ol><li>Portal-ing-http</li><li>Portal-ing-custom</li></ol>    |                                                                | Handles domain-specific routing with custom annotations and TLS configurations |
| 6    | Service           |           | <ol><li>portalbundle-svc-http</li><li>portalbundle-svc-headless</li></ol> |                                                                |                                                                          |
| 6    | PodDisruptionBudget |         | portalbundle-pdb                 | If tier is "paid"                                              |                                                                          |
| 7    | StatefulSet       |           | portalbundle-sts                 |                                                                |                                                                          |
| 8    | RoleBinding       | devportal | Devportal-read-resource           | Created when readRole is set to "enabled"                      | Grants permission defined in roles under roleRef                         |
| 9    | Role              | devportal | devportal-read-resource           |                                                                | Role definition designed to grant specific permissions to resources within a namespace, as defined by the RoleBinding |

## External Secrets

| S.No | Name                       | Consumer                                                       | Key                                                         |
| ---- | -------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------- |
| 1    | Portal-secret              | dr-restore job, elasticsearch-create-repo job, portalbundle-sts | <ol><li>OPSGENIE_AUTH_KEY - DPO Opsgenie Auth key</li><li>AZURE_STORAGE_ACCOUNT – Dpo Azure blob storage account</li><li>AZURE_STORAGE_KEY – Dpo Azure blob storage key</li><li>KC_TM_POWER_CLIENT_ID - Dpo TM's Power user Keycloak Client ID</li><li>KC_TM_POWER_CLIENT_SECRET - Dpo TM's Power user Keycloak Client Secret</li><li>FROM_ADDR – From Mail address</li><li>TO_ADDR – Mail id to be sent to</li><li>EMAIL_USER – User of the email id</li><li>EMAIL_PASSWORD – Password for the email</li><li>EMAIL_HOST – Host registered on the email</li><li>CLOUD_PROVIDER – Cloud provider such as AWS or Azure</li><li>DEFAULT_APP_PASSWORD – Default password for the app</li><li>SMTP_PASSWORD – SMTP User’s password</li><li>SMTP_USERNAME – Username of SMTP user</li><li>MASTER_APP_PASSWORD</li></ol> |
| 2    | portal-nonreplicate-secret | dr-restore job                                                 | REGION – eg: us-west2                                       |
| 3    | dpo-es-credentials         | elasticsearch-create-repo job, portalbundle-sts                | <ol><li>ANALYTICS_ES_PAID_USERNAME – eg:testes</li><li>ANALYTICS_ES_PAID_PASSWORD</li><li>ANALYTICS_ES_FFE_USERNAME - eg:testes</li><li>ANALYTICS_ES_FFE_PASSWORD</li><li>CORE_ES_PAID_USERNAME - eg:testes</li><li>CORE_ES_PAID_PASSWORD</li><li>CORE_ES_FFE_USERNAME - eg:testes</li><li>CORE_ES_FFE_PASSWORD</li><li>CLOUD_ANALYTICS_ELASTICSEARCH_USERNAME - eg:testes</li><li>CLOUD_ANALYTICS_ELASTICSEARCH_PASSWORD</li><li>SPRING_ELASTICSEARCH_USERNAME - eg:testes</li><li>SPR
