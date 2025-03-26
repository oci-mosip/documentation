# OCI Installation Guidelines

### Overview

* MOSIP modules are deployed in the form of microservices in a Kubernetes cluster.
* VPN is required for MOSIP operations. The implementer is free to choose the appropriate vpn.
* It is also used for on-the-field registrations.
* MOSIP uses OCI load balancers for:
  * SSL termination
  * Reverse Proxy
  * CDN/Cache management
  * Loadbalancing
* In this deployment setup V3, we have one [OKE](https://www.oracle.com/au/cloud/cloud-native/kubernetes-engine/) cluster
  * MOSIP Cluster - This cluster runs all the MOSIP components and certain third party components to secure the cluster, API’s and Data.
    * [MOSIP External Components](https://github.com/mosip/mosip-infra/blob/v1.2.0.1-B1/deployment/v3/external/README.md#mosip-external-components)
    * [Mosip Services](https://github.com/mosip/mosip-infra/blob/v1.2.0.1-B1/deployment/v3/mosip/README.md#mosip-services)
  * ArgoCD - The deployment and gitops is handled by ArgoCD(https://argo-cd.readthedocs.io/en/stable/ )

### Deployment Repos
* [mosip-infra](https://github.com/oci-mosip/mosip-gitops) : contains deployment scripts to run charts in defined sequence.
* [mosip-config](https://github.com/mosip/mosip-config/tree/v1.2.0.1-B1) : contains all the configuration files required by the MOSIP modules.
* [mosip-helm](https://github.com/mosip/mosip-helm/tree/v1.2.0.1-B1) : contains packaged helm charts for all the MOSIP modules.

### Pre-requisites:

#### Hardware Requirements

VM’s required have any Operating System and can be selected as per convenience.\
In this installation guide, we are referring to `Oracle Linux` throughout.

|   | **Purpose**                         | **vCPU’s** | **RAM** | **Storage (HDD)** | **no. of VM’s** | 
| - | ----------------------------------- | ---------- | ------- | ----------------- | --------------- | 
| 1 | Bastion Host                        | 2          | 4 GB    | 50 GB             | 1               | 
| 2 | Operator Host                       | 2          | 4 GB    | 50 GB             | 1               | 
| 3 | Mosip Cluster nodes (EKS managed)   | 8          | 32 GB   | 100 GB            | 6               | 

#### Network Requirements

* All the VM's should be able to communicate with each other.
* Need stable Intra network connectivity between these VM's.
* All the VM's should have stable internet connectivity for docker image download (in case of local setup ensure to have a locally accesible docker registry).
* During the process, we will be creating two loadbalancers as mentioned in the first table below:
* Server Interface requirement as mentioned in the second table:

| **Loadbalancer**                         | **Purpose**                                                                                                                                                                                                                   |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Public loadbalancer MOSIP cluster        | <p>This will be used to access below mentioned services:</p><ul><li>Pre-registration</li><li>Esignet</li><li>IDA</li><li>Partner management service api’s</li><li>Mimoto</li><li>Mosip file server</li><li>Resident</li></ul> |
| Private loadbalancer MOSIP cluster       | <p>This will be used to access all the services deployed as part of the setup inclusing external as well as all the MOSIP services.</p><p>Note: access to this will be restricted only with vpn access </p>        |

|   | **Purpose VM**         | **Network Interfaces**                                                                                                                                                                                                                                                                            |
| - | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | Bastion Host | <ul><li>One Private interface: that is on the same network as all the rest of nodes. (Eg: inside local NAT Network )</li><li>One public interface: Either has a direct public IP, or a firewall NAT (global address) rule that forwards traffic on 51820/udp port to this interface ip.</li></ul> |
| 2 | Operator Host | <ul><li>One Private interface: that is on the same network as all the rest of nodes. (Eg: inside local NAT Network )</li></ul> |

#### DNS Requirements

DNS zone management will be done in OCI [DNS](https://www.oracle.com/au/cloud/networking/dns/) service


|    | **Domain name**                                                     | **Mapping details**                    | **Purpose**                                                                                                                                                                                                                                     |
| -- | ------------------------------------------------------------------- | -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | [landing-page.sandbox.xyx.net](http://landing-page.sandbox.xyx.net)                           | Private Load balancer of MOSIP cluster | Index page for links to different dashboards of Mosip env. (This is just for reference, Please do not expose this page in a real production or uat environment)                                                                                 |
| 2  | [api-internal.sandbox.xyz.net](http://api-internal.sandbox.xyz.net) | Private Load balancer of MOSIP cluster | Internal API’s are exposed through this domain. They are accessible privately over wireguard channel                                                                                                                                            |
| 3  | [api.sandbox.xyx.net](http://api.sandbox.xyx.net)                   | Public Load balancer of MOSIP cluster  | All the API’s that are publically usable are exposed using this domain.                                                                                                                                                                         |
| 4  | [prereg.sandbox.xyz.net](http://prereg.sandbox.xyz.net)             | Public Load balancer of MOSIP cluster  | Domain name for Mosip’s pre-registration portal. The portal is accessible publicly.                                                                                                                                                             |
| 5  | [activemq.sandbox.xyx.net](http://activemq.sandbox.xyx.net)         | Private Load balancer of MOSIP cluster | Provides direct access to activemq dashboard. Its limited and can be used only over wireguard                                                                                                                                                   |
| 6  | [kibana.sandbox.xyx.net](http://kibana.sandbox.xyx.net)             | Private Load balancer of MOSIP cluster | Optional instalation. Used to access kibana dashboard over wireguard                                                                                                                                                                            |
| 7  | [regclient.sandbox.xyz.net](http://regclient.sandbox.xyz.net)       | Private Load balancer of MOSIP cluster | Regclient can be downloaded from this domain. It should be used over wireguard.                                                                                                                                                                 |
| 8 | [admin.sandbox.xyz.net](http://admin.sandbox.xyz.net)               | Private Load balancer of MOSIP cluster | Mosip’s admin portal is exposed using this domain. This is an internal domain and is restricted to access over wireguard                                                                                                                        |
| 9 | [minio.sandbox.xyx.net](http://minio.sandbox.xyx.net) | Private Load balancer of MOSIP cluster | Optional- This domain is used to access the object server. Based on the object server that you choose map this domain accordingly. In our reference implementation Minio is used and this domain lets you access Minio’s Console over wireguard |
| 10 | [kafka.sandbox.xyz.net](http://kafka.sandbox.xyz.net)               | Private Load balancer of MOSIP cluster | Kafka UI is installed as part of the Mosip’s default installation. We can access kafka ui over wireguard. Mostly used for administrative needs.                                                                                                 |
| 11 | [iam.sandbox.xyz.net](http://iam.sandbox.xyz.net)                   | Private Load balancer of MOSIP cluster | Mosip uses an Openid connect server to limit and manage access across all the services. The default installation comes with Keycloak. This domain is used to access the keycloak server over wireguard                                          |
| 12 | [postgres.sandbox.xyz.net](http://postgres.sandbox.xyz.net)         | Private Load balancer of MOSIP cluster | This domain points to the postgres server. You can connect to postgres via port forwarding over wireguard                                                                                                                                       |
| 13 | [pmp.sandbox.xyz.net](http://pmp.sandbox.xyz.net)                   | Public Load balancer of MOSIP cluster  | Mosip’s partner management portal is used to manage partners accessing partner management portal over wireguard                                                                                                                                 |
| 14 | [resident.sandbox.xyz.net](http://resident.sandbox.xyz.net)         | Public Load balancer of MOSIP cluster  | accessident resident portal publically                                                                                                                                                                                                          |
| 15 | [esignet.sandbox.xyz.net](http://idp.sandbox.xyz.net)               | Public Load balancer of MOSIP cluster  | accessing IDP over public                                                                                                                                                                                                                       |
| 16 | [smtp.sandbox.xyz.net](http://smtp.sandbox.xyz.net)                 | Private Load balancer of MOSIP cluster | Accessing mock-smtp UI over wireguard                                                                                                                                                                                                           |

**Note:**

* Only proceed to DNS mapping after the ingressgateways are installed and the load balancer is already configured.
* The above table is just a placeholder for hostnames, the actual name itself varies from organisation to organisation.

#### Certificate requirements

As only secured `https` connections are allowed via nginx server, you will need the below mentioned valid ssl certificates:

* One valid wildcard ssl certificate related to domain used for accessing MOSIP cluster which will be created using [cert-manager](https://cert-manager.io/). In above e.g. \*.[sandbox.xyz.net](http://sandbox.xyz.net/) is the similiar example domain.

#### Prerequisite for complete deployment in Personal Computer
-   Install Docker
-   Generate admin user api keys as per [OCI Docs](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm#two)
-   Capture the values for
    - user_id
    - tenancy_id
    - fingerprint
    - region
    - private_key
-   Create a customer secret key for storing terraform state in remote bucket as per [OCI Docs](https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/s3compatibleapi.htm#usingAPI)
-   Capture the values for
    - s3 access key
    - s3 secret key
    - s3 region
    - s3 comptability api endpoint
-   Create bucket for storing terraform state files
    - capture the name of the bucket
-   Create DNS zone in OCI and delegate the zone control
-   Create software Vault and master vault encryption key in OCI


### Installation

#### Deployment diagram

![](../../../.gitbook/assets/oci_deployment_architecture.png)


**Setup VPN:**

TODO

**Setup VPN Client in your laptop**

TODO

