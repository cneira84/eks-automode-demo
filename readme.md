# EKS Auto Mode - Demo Nerdearla 2025

## Despliegue de un cluster EKS utilizando Auto Mode

### Prerequisito: Crear roles necesarios de EKS Auto Mode
EKS Auto Mode requiere dos roles en su creación.
- El primero permite realizar tareas de rutina sobre el cluster. Más informació en documentación de [Amazon EKS Auto Mode cluster IAM role](https://docs.aws.amazon.com/eks/latest/userguide/auto-cluster-iam-role.html)
- El segundo permite la operación sobre los nodos. Más información en la documentación de [Amazon EKS Auto Mode node IAM Role](https://docs.aws.amazon.com/eks/latest/userguide/auto-create-node-role.html)

Utilizar el siguiente comando para desplegar el cloudformation que crea ambos roles

```
export STACK_ROLES=automode-roles
aws cloudformation create-stack --stack-name $STACK_ROLES --template-body file://prereqs/eks-automode-roles-nerdearla.yaml --capabilities CAPABILITY_NAMED_IAM
```

Copiar el output del cloudformation o utilizar el siguiente comando para exportarlo:

```
export CLUSTER_ROLE_ARN=$(aws cloudformation describe-stacks --stack-name $STACK_ROLES --query 'Stacks[0].Outputs[?OutputKey==`ClusterRoleArn`].OutputValue' --output text)
export NODE_ROLE_ARN=$(aws cloudformation describe-stacks --stack-name $STACK_ROLES --query 'Stacks[0].Outputs[?OutputKey==`NodeRoleArn`].OutputValue' --output text)
```

### Crear el cluster EKS Auto Mode con eksctl

Antes de iniciar, verificar que tenemos el [set-up terminado](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html). Esto solo es necesario si nunca usaste kubectl o eksctl.

Antes de correr los comandos para la creación, actualizar los roles en el manifiesto utilizado por eksctl que se encuentra en `cluster/eksctl-cluster-demo-nerdearla.yaml`

```
export CLUSTER_NAME=cluster-nerdearla
eksctl create cluster -f ./cluster/eksctl-cluster-demo-nerdearla.yaml
```

### Deploy ingress-class
Para poder publicar nuestra app a internet, necesitamos previamente crear un IngressClass. Este objeto permite la interacción con AWS Load Balancer Controller que ya se encuentra funcionando como parte de EKS Auto Mode. 
Vamos a desplegar dos objetos:
1. `IngressParams` que contiene los parametros que utilizará IngressClass como `scheme: internet-facing`
2. `IngressClass` que será la clase que utilizaremos en la definición del ingress de nuestra app.

```
kubectl apply -f ./cluster/ingress-class.yaml
```

## Deploy de una sample app
Vamos a desplegar una app de ejemplo, que se utiliza como base de demostración para otros workshops y demos: [AWS Container Retail Sample](https://github.com/aws-containers/retail-store-sample-app)

Solo para el caso de UI vamos a utilizar valores custom. Se puede encontrar más información sobre el UI Helm Chart [aquí](https://github.com/aws-containers/retail-store-sample-app/tree/main/src/ui/chart)

Para utilizar la imagen pública, es necesario loguearnos primero al repositorio:
```
aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
```

Luego podemos desplegar los componentes de la app. Si no tenemos Helm instalado, podemos seguir estos [pasos para desplegarlo](https://helm.sh/docs/helm/helm_install/)
```
helm upgrade -i retail-store-app-catalog oci://public.ecr.aws/aws-containers/retail-store-sample-catalog-chart --version 1.3.0 --hide-notes
helm upgrade -i retail-store-app-orders oci://public.ecr.aws/aws-containers/retail-store-sample-orders-chart --version 1.3.0 --hide-notes
helm upgrade -i retail-store-app-carts oci://public.ecr.aws/aws-containers/retail-store-sample-cart-chart --version 1.3.0 --hide-notes
helm upgrade -i retail-store-app-checkout oci://public.ecr.aws/aws-containers/retail-store-sample-checkout-chart --version 1.3.0 --hide-notes
helm upgrade -i retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version 1.3.0 -f ./values-for-apps/values-ui.yaml --hide-notes
```

### Validar la app
En este momento, debería desplegarse de forma automática el ALB. Para verificarlo podemos ejecutar estos comandos

```
export ALB_URL=$(kubectl get ingress retail-store-app-ui -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
# wait for the alb to become active
aws elbv2 wait load-balancer-available --load-balancer-arns $(aws elbv2 describe-load-balancers --query 'LoadBalancers[?DNSName==`'"$ALB_URL"'`].LoadBalancerArn' --output text)

echo "Your application is available at: http://${ALB_URL}"
```
Ahora podemos consultar la URL obtenida desde un browser o ejecutar un `curl http://${ALB_URL}`

## Ajustes y optimizaciones de costos y resiliencia
### Ajustando el Nodepool para usar instancias spot
Con el fin de optimizar costos, podemos crear nuestra propia definición de NodePool e utilizar instancias spot.
Para monitorear la ejecución vamos a utilizar [eks-node-viewer](https://github.com/awslabs/eks-node-viewer) (este paso es opcional)

En una pestaña abrir `eks-node-viewer` para monitorear los nodos. Esta tool también permite ver el costo de cada nodo.

Desplegar el nuevo nodepool

```
kubectl apply -f ./nodepools/nodepool-spot.yaml
```
Ahora vamos a observar como Karpenter reajusta los nodos a una instancia más costo efectiva utilizando spot.

### Ajustando mi deployment para tener un spread topology
Con el objetivo de poder realizar una distribución que incremente la resiliencia a nivel de AZs, podemos definir dentro de nuestro deployment el comportamiento deseado. Corramos el siguiente update a la UI para poder verlo en funcionamiento

```
helm upgrade -i retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version 1.3.0 -f ./values-for-apps/values-ui-spread.yaml --hide-notes
```

Verificar la distribución ejecutando
```
kubectl get pods -o wide
```

## Clean-up

1. Eliminar la app
```
helm uninstall retail-store-app-catalog
helm uninstall retail-store-app-orders
helm uninstall retail-store-app-carts
helm uninstall retail-store-app-checkout
helm uninstall retail-store-app-ui
```

2. Eliminar cluster
```
eksctl delete cluster $CLUSTER_NAME`
```

3. Eliminar stack de creación de roles
```
aws cloudformation delete-stack --stack-name automode-roles`
```

## Tooling
https://github.com/awslabs/eks-node-viewer

## Referencias
https://docs.aws.amazon.com/eks/latest/userguide/automode-get-started-eksctl.html





