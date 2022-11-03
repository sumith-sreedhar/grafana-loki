Customer Requirement:

Grafana-Loki Multi tenant setup:

a) One account will be the AWS management account where you can install the full loki stack (Loki, Grafana, promtail). <br />
b) The other two AWS accounts should have only promtail running in the EKS clusters, scraping the sample app logs and reporting to the central loki running in the management account cluster. Then that loki instance can be used as a data source for Grafana. <br />
c) Then the RBAC will be implemented in Grafana to allow tenants to only see their logs when they log in to Grafana but allow me as the admin to see the entire thing.<br />

**************************************************************************************************************************************************************

Steps:

1) Configure EKS cluster<br />

   #eksctl create cluster --name loki-promtail --region us-east-1 --managed

2) Install the AWS Load Balancer Controller add-on in the main EKS cluster. Ref: https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html <br />

3) Install Grafana & loki as seperate services using helm <br />

   #helm upgrade --install loki --namespace=monitoring --set grafana.enabled=false,promtail.enabled=true  grafana/loki-stack --values loki-values.yaml

   #helm install grafana grafana/grafana --namespace=monitoring

4) Grafana load balancer becomes classic ELB. Loki use NLB using the following annotation in loki service:

    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: instance
    service.beta.kubernetes.io/aws-load-balancer-scheme: internal
    service.beta.kubernetes.io/aws-load-balancer-type: nlb-ip

5) Create a Private link with the tenant cluster sharing NLB. NLB private dns can be obtained from tenant vpc endpoint. Create an ec2 instance in tenant subnet and try curl that IP.
Eg: curl -G -s http://10.0.12.118:3100/loki/api/v1/labels | jq


Considrations:

a) Ensure that auth_enabled: true is added in Loki secret/configmap.

------------------------------------------------------------------------------------------------------------------------------------------------------------------

Configure the tenant cluster:

1) Run the following command as the other aws account:

   #eksctl create cluster --name Tenant1 --region us-east-1 --managed

2)  Add cluster as a VPC endpoint to the endpoint service configured in main  EKS Cluster.

3) Install Promtail agent:

   #helm upgrade  promtail loki/promtail -n promtail -f promtail-tenant1-values.yaml

Tenant auth details are defined in promtail-tenant1-values.yaml . loki url IP needs to be the tenant endpoint privte dns IP


--------------------------------------------------------------------------------------------------------------------------------------------------------------

Grafana Setp:

Login Grafana as admin user.
Create Grafana Organizations and Add Loki data source (url is http://loki:3100 in this setup). Also enable Basic Auth and enter the credentials defined in promtail-tenant1-values.yaml
Enter X-Scope-OrgID value (defined in promtail-tenant1-values.yaml) in Custom HTTP Headers
Save & Test


***********************************************************************************************************************************************************************
Referrences:

https://itnext.io/multi-tenancy-with-loki-promtail-and-grafana-demystified-e93a2a314473 <br />
https://grafana.com/blog/2020/07/21/loki-tutorial-how-to-send-logs-from-eks-with-promtail-to-get-full-visibility-in-grafana/ <br />
https://bluelight.co/blog/how-to-install-grafana-loki <br />
