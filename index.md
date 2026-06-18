# DNS Entry Migration

## Introduction

The following describes a DNS migration using the Azure Traffic Manager to migrate your DNS entries from Transformer 1 service instances to Transformer 2. It enables you to quickly react in a central resource on problems that might come up during the migration, and also offers weighted migration traffic scenarios.

Azure Traffic Manager is a DNS-based traffic load balancer. It allows to distribute traffic to your public facing applications across the global Azure regions.

## Prerequisites

Copy the TLS certificates for host names that need migration into the target namespace. That way you do not need to create them during the migration. If you skip this step you might have a longer downtime until your service is up again, because clients will only be able to validate your service, when the new certificate has been validated.

Hint

Do not use the cert-manager annotation as part of your ingress, when you migrate. It is safer during migration to have the certificate under your direct control.

## Migration of platform owned DNS zone entries

The following applies to DNS entries in these DNS zones:

*.apps.mega.cariad.cloud  
*.apps.mega.vwautocloud.cn  

## Preparations

Make sure, the DNS requests for clients have a shorter lifetime during the migration. Default DNS record lifetime is 3600s. If some clients query your Service DNS entry just before you change and you change quickly, they could be offline for up to an hour. This can be changed by an external-dns annotation:

```yaml
external-dns.alpha.kubernetes.io/hostname: your-service.apps.mega.cariad.cloud
external-dns.alpha.kubernetes.io/target: nginx-waf.paris.clusters.mega.cariad.cloud
external-dns.alpha.kubernetes.io/ttl: "60"

Create a Traffic Manager profile, which allows to later switch the traffic towards your Transformer 2 service instance.

Retrieve the IP address of the Application Gateway or nginx-loadbalancer that currently handles your Transformer 1 service
dig your-service.apps.mega.cariad.cloud +short
nginx-waf.paris.clusters.mega.cariad.cloud.
20.23.99.247

Then create a new Traffic Manager Profile in your Transformer 2 subscription and add an endpoint for your Transformer 1 service. ./images/

Add an endpoint for your Transformer 2 instance in your Traffic Manager profile. As the endpoint IP you use the loadbalancer IP of the Transformer 2 cluster, that you can retrieve like this:

dig argocd.preview.eu-dev.mega.cariad.cloud +short 
135.116.10.134

Configure the DNS entry TTL in Traffic Manager to be 60s. Also configure a health probe in the Traffic Manager Configuration view that works for both service instances. Afterwards make sure that your endpoints are shown as online. ./images/

Now you can test that both services are reachable via the Traffic Manager URL. Note that below curl command accepts an invalid certificate (-k). This is necessary, because by providing the target IP (technically via an DNS A record) the Traffic Manager transparently lets your client connect to your service. Your service shows the certificate configured in ingress, but not a valid certificate for the Traffic Manager site that curl believed to call. The below commands will show different results over time, because per default both endpoints have the same weight:

dig your-service.apps.mega.cariad.cloud +short
your-traffic-manager-profile.trafficmanager.net.
20.23.99.247
curl -v -k https://your-trafficmanager-profile.trafficmanager.net

When you are sure, both services are generally reachable, disable the Transformer 2 service endpoint. This way you can make sure in the next step, that all clients will continue to use your Transformer 1 service.

Activate Traffic Manager for Transformer 1 Service

When only your Transformer 1 endpoint is enabled in Traffic Manager you can proceed with changing the CNAME entry of your Transformer 1 service to point towards Traffic Manager, by adding external-dns annotations on the ingress resource. This will not result in downtime, as it changes the DNS entry for the clients in a way, that the end result after following the CNAME chain will still be your old service.

external-dns.alpha.kubernetes.io/hostname: your-service.apps.mega.cariad.cloud
external-dns.alpha.kubernetes.io/target: your-trafficmanager-profile.trafficmanager.net
external-dns.alpha.kubernetes.io/ttl: "60"

Afterwards traffic continues to be served from your Transformer 1 service, but the Traffic Manager should be shown as a CNAME. Check the DNS entry like this:

dig your-service.apps.mega.cariad.cloud +short
your-trafficmanager-profile.trafficmanager.net.
20.23.99.247
Switch over to Transformer 2 service instance

After the previous steps, the actual migration can be done. It can be done gradually, by adjusting the weights of the endpoints as you need it. When you have set the weights, you can enable the Transformer 2 service. For the time of the DNS TTL you configured in the Traffic Manager, clients will try call both endpoints. That is why the TTL should be configured with a low value to be able to react fast in case of problems.

Then try to connect to your service:

TARGET_IP=135.116.10.134   # Put the IP-address of the target cluster nginx here, e.g. you can get that from the ArgoCD instance. ArgoCD uses the same ip address.
HOST_NAME=                 # The DNS hostname, that needs to be migrated

curl --resolve $HOST_NAME:443:$TARGET_IP https://$HOST_NAME/ -v

When everything works as expected, deactivate the Transformer 1 service.

To remove the Traffic Manager from your setup to save costs, configure external-dns to route directly to the Transformer 2 loadbalancer, and have a standard DNS TTL of 3600s again. The following creates an A record for your Transformer 2 service:

external-dns.alpha.kubernetes.io/hostname: your-service.apps.mega.cariad.cloud
external-dns.alpha.kubernetes.io/target: 135.116.10.134
external-dns.alpha.kubernetes.io/ttl: "3600"

Observe the traffic to be stable for some time then you can remove the Traffic Manager Profile resource completely.

Special cases

This applies to a subset of these DNS zones, where the DNS record used contains exactly the Transformer 1 productname and the Transformer 1 stagename:

-.apps.mega.cariad.cloud
-.apps.mega.vwautocloud.cn

In case your application uses ingresses with hostnames like this, please check beforehand:

Is there a DNS txt entry for your ingress? This can be checked like this:

dig TXT external-dns-api.<product>-<stage>.apps.mega.cariad.cloud. +short
dig TXT external-dns-cname.<product>-<stage>.apps.mega.cariad.cloud. +short

If either of these commands deliver a result referencing your ingress object (e.g. "heritage=external-dns,external-dns/owner=kyiv,external-dns/resource=ingress/sprinarg-par/sprinarg-service-ingress"), the DNS record has been created by external-dns. This is the easy case, you can treat it like described in the standard case of a platform owned DNS zone entry above.

If you do not find a TXT record for your ingress hostname, the entry has been created during the product stage creation process automatically, and is still in platform hands. The transfer process is that you request the platform team to remove the DNS record and directly afterwards you create a new entry in Transformer 2 using the ingress resource with external-dns annotation. This will come with a little downtime where new clients who do not have your DNS entry in cache will have to wait for the new DNS entry. That is why this needs to be done in a joint session with support to make sure the handover takes place as smooth as possible.

Please create a service desk ticket, stating that:

We want to migrate our DNS record <YOUR_DNS_RECORD> towards Transformer 2

it has not been created by external-dns, therefore we need your platform support
to minimize customer downtime we would like you to remove the DNS entry in a joint online session with us, where we afterwards create the ingress in Transformer 2 again using external-dns.
to further minimize client downtimes, please set the TTL of the current DNS record to 60s ahead of this session,
Migration of *.cariad.digital domains

The process can be the same as above described for platform owned domains with one exception: instead of external-dns annotations you perform the necessary changes manually in your own child DNS zone.