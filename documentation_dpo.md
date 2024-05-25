Portal-ingress\
Overview

Kubernetes Resources

Below are the list of Kubernetes resources as part of this helm chart
that would get created

+----+--------+---------+--------+-----------------------+-----------+
| S. | Kind   | Na      | Re     | Conditional Resources | Comments  |
| No |        | mespace | source |                       |           |
+====+========+=========+========+=======================+===========+
| 1  | i      |         | i      | isTlsEnabled          |           |
|    | ngress |         | ngress |                       |           |
|    |        |         |        | envRole is "hub" and  |           |
|    |        |         |        | cloudflare is enabled |           |
|    |        |         |        |                       |           |
|    |        |         |        | isIng                 |           |
|    |        |         |        | ressAnnotationDefined |           |
|    |        |         |        |                       |           |
|    |        |         |        | isCdmIng              |           |
|    |        |         |        | ressAnnotationDefined |           |
+----+--------+---------+--------+-----------------------+-----------+
| 2  | Job    | Spoke   | 1.     | envRole is "spoke"    |           |
|    |        | -system | Spoke- | and                   |           |
|    |        |         | custom | len(                  |           |
|    |        |         | -domai | customDomain.domains) |           |
|    |        |         | n-ngin | ne 0                  |           |
|    |        |         | x-conf |                       |           |
|    |        |         | ig-job |                       |           |
|    |        |         | :      |                       |           |
|    |        |         | dpo-c  |                       |           |
|    |        |         | dmncj\ |                       |           |
|    |        |         | \      |                       |           |
|    |        |         | \      |                       |           |
|    |        |         | \      |                       |           |
|    |        |         | 2.     |                       |           |
|    |        |         | spoke- |                       |           |
|    |        |         | delete |                       |           |
|    |        |         | -nginx |                       |           |
|    |        |         | -confi |                       |           |
|    |        |         | g-job: |                       |           |
|    |        |         | dp     |                       |           |
|    |        |         | o-dncj |                       |           |
+----+--------+---------+--------+-----------------------+-----------+
| 3  | con    | Spoke   | Spok   | envRole is "spoke"    | These     |
|    | figmap | -system | e-ngin | and                   | c         |
|    |        |         | x-conf | len(                  | onfigmaps |
|    |        |         | igmap: | customDomain.domains) | have      |
|    |        |         |        | ne 0                  | product   |
|    |        |         |        |                       | specific  |
|    |        |         |        |                       | nginx     |
|    |        |         |        |                       | confi     |
|    |        |         |        |                       | gurations |
+----+--------+---------+--------+-----------------------+-----------+

Portal-storage

Overview

  ---------- ---------------- ------------ ------------------------- --------------- ----------
  1          RoleBinding      devportal    Devportal-read-resource   readRole is     
                                                                     enabled         

  2          Role             devportal    devportal-read-resource   cloudProvider   
                                                                     is aws          

  3          ServiceAccount   management   aksreaduser               Cloudprovider   
                                                                     not aws\        
                                                                     \               
                                                                     readRole is     
                                                                     enabled         

  4          Group                         eksreaduser               Cloudprovider   
                                                                     not aws\        
                                                                     \               
                                                                     readRole is     
                                                                     enabled         

  4          StorageClass                  devportal-block-storage                   
  ---------- ---------------- ------------ ------------------------- --------------- ----------
