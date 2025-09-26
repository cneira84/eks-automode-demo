# EKS Auto Mode - Demo Nerdearla 2025

## Despliegue de un cluster EKS utilizando Auto Mode

### Crear roles necesarios de EKS Auto Mode (Prerequisito)
Se puede encontrar más información de la creación en el siguiente link.

```
export STACK_NAME_ROLES=automode-roles
export CLUSTER_NAME=cluster-nerdearla
```

`aws cloudformation create-stack --stack-name $STACK_NAME_ROLES --template-body file://prereqs/eks-automode-roles-nerdearla.yaml --capabilities CAPABILITY_NAMED_IAM`

Copiar el output de los roles o utilizar el siguiente comando para exportarlo:

```
export CLUSTER_ROLE_ARN=$(aws cloudformation describe-stacks --stack-name $STACK_NAME_ROLES --query 'Stacks[0].Outputs[?OutputKey==`ClusterRoleArn`].OutputValue' --output text)
export NODE_ROLE_ARN=$(aws cloudformation describe-stacks --stack-name $STACK_NAME_ROLES --query 'Stacks[0].Outputs[?OutputKey==`NodeRoleArn`].OutputValue' --output text)
```

### Crear el cluster utilizando EKS Auto Mode

Actualizar el archivo eksctl-cluster-demo-nerdearla.yaml

`eksctl create cluster -f ./cluster/eksctl-cluster-demo-nerdearla.yaml`

### Deploy ingress-class

Desplegar ingress-params e ingress-class para exponer la app a internet

`kubectl apply -f ./cluster/ingress-class.yaml`


### Deploy sample app

Vamos a desplegar una app en containers de ejemplo: [Retail Demo Store](https://github.com/aws-containers/retail-store-sample-app)

Solo para el caso de UI vamos a utilizar valores custom. Se puede encontrar más información sobre el UI Helm Chart [aquí](https://github.com/aws-containers/retail-store-sample-app/tree/main/src/ui/chart)


```
helm upgrade -i retail-store-app-catalog oci://public.ecr.aws/aws-containers/retail-store-sample-catalog-chart --version 1.3.0 --hide-notes
helm upgrade -i retail-store-app-orders oci://public.ecr.aws/aws-containers/retail-store-sample-orders-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes
helm upgrade -i retail-store-app-carts oci://public.ecr.aws/aws-containers/retail-store-sample-cart-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes
helm upgrade -i retail-store-app-checkout oci://public.ecr.aws/aws-containers/retail-store-sample-checkout-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes
helm upgrade -i retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} -f ~/environment/values-ui.yaml --hide-notes
```
In case of 

### Validar la app

Exportamos la URL del ALB

```
export ALB_URL=$(kubectl get ingress retail-store-app-ui -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
# wait for the alb to become active
aws elbv2 wait load-balancer-available --load-balancer-arns $(aws elbv2 describe-load-balancers --query 'LoadBalancers[?DNSName==`'"$ALB_URL"'`].LoadBalancerArn' --output text)

echo "Your application is available at: http://${ALB_URL}"
```
## Referencias

https://docs.aws.amazon.com/eks/latest/userguide/automode-get-started-eksctl.html

## Clean-up

eliminar cluster
`eksctl delete cluster $CLUSTER_NAME`

eliminar stack
`export STACK_NAME_ROLES=automode-roles

