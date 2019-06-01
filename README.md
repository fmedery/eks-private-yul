# How does Amazon EKS work?

![EKS](https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Logo Title Text 1")
# Introduction
* le VPC est déja crée et comprend:
  * deux subnet privés
  * deux subnet publics
* EKS n'exposera pas de service à internet et on utilise que les subnets privés
* Le role de service `eksServiceRole` peut-être le même pour tous les clusters EKS du compte
* EKS_SUBNET_IDS ne doit contenir que les réseau privés
* Il faut attendre la création de EKS avant la création des nodes

# Prérequis sur l'ordinateur de l'opérateur 
* Logiciel installés et disponible via $PATH:
  * awscli
  * kubectl
  * aws-iam-authentificator
* awscli config file:
  * key
  * secret
  * ca-central-1
* L'usager IAM doit-être capable de créer des ressources dans le compte:
  * AIM role
  * EKS cluster
  * EC2
  * ELB
  * auto scaling group
* Ajouter les tags suivants sur les réseaux privé:
  ```
  KEY: kubernetes.io/role/internal-elb
  VALUE: 1 
  ```

# Création du cluster EKS
```sh
# VARIABLES: edition obligatoire
export EKS_CLUSTER_NAME="eks-cluster" # nom du cluster EKS 
export EKS_NODE_TYPE="t2.small" # type de machine 
export EKS_NODE_MIN="3" # nombre minimum de nodes
export EKS_NODE_MAX="3" # nombre maximum de nodes
export EKS_VPC_ID="vpc-0403e2434c78a7e9e" # https://ca-central-1.console.aws.amazon.com/vpc/home?region=ca-central-1#vpcs:sort=VpcId
export EKS_SECURITY_GROUPS="sg-0e5b8ed40b0f9d0a5" #
export EKS_SUBNET_IDS="subnet-0d94577440a441dd0,subnet-07816231a0ead0694" #  https://ca-central-1.console.aws.amazon.com/vpc/home?region=ca-central-1#subnets:sort=SubnetId

# VARIBLES: edition facultative
export EKS_WORKER_AMI=ami-079df7c59d89ba556 # AMI optimisée pour EKS dans la region ca-central-1
export REGION=ca-central-1 # Région 
export EKS_SERVICE_ROLE_NAME="eksServiceRole" # nom du role de service EKS
export WORKER_STACK_NAME="${EKS_CLUSTER_NAME}-worker-nodes" # nom de la stack pour la création des nodes
export EKS_NODE_GROUP_NAME="${EKS_CLUSTER_NAME}-worker-group" # nom du nodegroup

# Création du role de service
aws --region ${REGION} cloudformation create-stack \
    --stack-name eks-service-role \
    --template-body file://amazon-eks-service-role.yaml \
    --capabilities CAPABILITY_NAMED_IAM

# Création de EKS
export EKS_SERVICE_ROLE=$(aws --region ${REGION} iam list-roles --query 'Roles[?contains(RoleName, `eksService`) ].Arn' --out text) # on va chercher le ARN
aws --region ${REGION} eks create-cluster \
    --name $EKS_CLUSTER_NAME \
    --role-arn $EKS_SERVICE_ROLE \
    --resources-vpc-config subnetIds=$EKS_SUBNET_IDS,securityGroupIds=$EKS_SECURITY_GROUPS


# La création de EKS va prendre environ 12 minutes 
 aws --region ${REGION} eks describe-cluster --name $EKS_CLUSTER_NAME --query 'cluster.status' # Vérification du status

# Création des nodes
export EKS_ENDPOINT=$(aws eks describe-cluster --name $EKS_CLUSTER_NAME --query cluster.endpoint)
export EKS_CERT=$(aws eks describe-cluster --name $EKS_CLUSTER_NAME --query cluster.certificateAuthority.data)

echo EKS_ENDPOINT: $EKS_ENDPOINT
echo ""
echo EKS_CERT: $EKS_CERT

  aws --region ${REGION} cloudformation create-stack \
      --stack-name $WORKER_STACK_NAME  \
      --template-body file://amazon-eks-nodegroup.yaml \
      --capabilities CAPABILITY_IAM \
      --parameters \
        ParameterKey=NodeInstanceType,ParameterValue=${EKS_NODE_TYPE} \
        ParameterKey=NodeImageId,ParameterValue=${EKS_WORKER_AMI} \
        ParameterKey=NodeGroupName,ParameterValue=${EKS_NODE_GROUP_NAME} \
        ParameterKey=NodeAutoScalingGroupMinSize,ParameterValue=${EKS_NODE_MIN} \
        ParameterKey=NodeAutoScalingGroupMaxSize,ParameterValue=${EKS_NODE_MAX} \
        ParameterKey=ClusterControlPlaneSecurityGroup,ParameterValue=${EKS_SECURITY_GROUPS} \
        ParameterKey=ClusterName,ParameterValue=${EKS_CLUSTER_NAME} \
        ParameterKey=Subnets,ParameterValue=${EKS_SUBNET_IDS//,/\\,} \
        ParameterKey=VpcId,ParameterValue=${EKS_VPC_ID} \
        ParameterKey=KeyName,ParameterValue=${WORKER_STACK_NAME}


export EKS_INSTANCE_ROLE=$(aws --region ${REGION} iam list-roles --query 'Roles[?contains(RoleName, `'${EKS_CLUSTER_NAME}'-worker-nodes`) ].Arn' --out text)

```