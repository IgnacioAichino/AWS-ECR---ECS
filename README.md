# Pipeline de GitLab para desplegar en ECS en AWS
## _GitFlow_
La idea principal detrás de GitFlow es aislar el trabajo en diferentes tipos de branches, lo que le permite adaptarse muy bien al proceso colaborativo que necesita un equipo de desarrollo. GitFlow está basado principalmente en tres branches que tienen una vida infinita:

**master:** contiene el código de producción.
**develop:** contiene el código que ha finalizado desarrollo.
**feature:** se crea a partir de develop cuando una nueva funcionalidad necesita ser desarrollada. Al finalizar el desarrollo se hace merge a develop nuevamente.

## Preparacion y fundamentos

Si establecemos una estrategia de ramas. Desarrollaremos el pipeline para lograr los deployments en ECS.
Este proyecto de ejemplo se utiliza docker-compose para automatizar el despliegue de la aplicacion.
Debemos contar con una cuenta de AWS en la cual crearemos un repositorio de imagenes de docker gracias al servicio de **ECR**

Este repositorio cuenta con un archivo **.yml** el cual tiene desarrollado el pipeline en GitLab para construir una imagen de docker de nuestra app y desplegarla en un ambiente de test y produccion con docker-compose en el servicio de **ECS** de AWS.

**ECS** es un servicio que nos permite ejecutar y administrar un cluster de contenedores Docker. 
Vamos a utilizar un cluster Fargate para asignar recursos a nuestro host que ejecutara el contenedor en el cluster.

Debemos contar con la informacion de URL del repositorio como tambien la region y definirlo como variable en nuestro pipeline:

```yaml
variables:
  REPOSITORY_URL:  repository in AWS
  REPOSITORY_REGION: us-east-1
```

Siguendo la estratigia de ramas mencionada nuestro pipeline tiene definido 4 stages: 
```yaml
stages:
  - prepare
  - build
  - publish 
  - deploy-test
  - deploy-prod
```

Tanto **prepare, build, publish, y deploy-test** tiene un _tag_ para que el pipeline ejecutre sobre la rama develop, y main de nuestro proyecto. En esta rama _develop_ se encuentra las nuevas funcionalidades del codigo en desarrollo por lo que los commits realizados sobre esta rama desencadenan en la ejecucion de estos stage para preparar el entorno de contruccion de la imagen, conectarnos a ECR donde alojaremos la imagen creada de nuestra app y luegos desplegemos en un ambiente de test.
Se muestran mas adelante las tareas de CI/CD en cada etapa.

La etapa de **deploy-prod** puede ser manual o automatica segun la estrategia que decidamos. En este caso sera manual y se ejecuta solo en la rama _main_ ya que siguiendo la estrategia de ramas, esta contendra el codigo de la app en produccion. La ejecucion del pipeline espera como argumento de entrada un valor de version de nuestra app a ser desplegada en un ambiente productivo.


#### _Requisitos_

- Instalar la CLI de AWS: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
Para este paso podemos automatizar la instalacion utilizando la imagen de AWS brindada por GitLab ya que estamos desarrolando un pipeline de GitLab.
https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-push-ecr-image.html.
De esta manera definimos en nuestro **stage: _prepare_** para que utilice la imagen mencionada y la tarea llevada adelante de registro de nuestra cuenta.

```yaml
Prepare:
  stage: prepare
  image: registry.gitlab.com/gitlab-org/cloud-deploy/aws-base:latest
  script:
    - echo "REGISTRY_PASS=$(aws ecr get-login-password --region "$REPOSITORY_REGION")"
```


### _Build and push_

Pasaremos en la siguiente etapa del pipeline a contruir la image de nuestra app mediante el Dockerfile que hayamos creado para nuestro proyecto.
```yaml
Build:
  stage: build
  image:
    name: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $REPOSITORY_URL/$FRONTEND_IMAGE:$IMAGE_TAG -f ./front/Dockerfile ./front
  needs:
    - Prepare
```

Una vez creada la imagen podremos loguearnos a nuestro repositorio y hacer **push** a nuestro repositorio ECR en AWS y disponer de la imagen creada para definir tasks en ECR.

```yaml
Publish:
  stage: publish
  image:
    name: docker:latest
  services:
    - docker:dind
  before_script:
    - echo $REGISTRY_PASS | docker login --username AWS --password-stdin $REPOSITORY_URL
  script:
    - docker tag $REPOSITORY_URL/$FRONTEND_IMAGE:$IMAGE_TAG $REPOSITORY_URL/$FRONTEND_IMAGE:latest
    - docker push $REPOSITORY_URL/$FRONTEND_IMAGE:$IMAGE_TAG
    - docker push $REPOSITORY_URL/$FRONTEND_IMAGE:latest
```

### Deployment 

Si el pipeline ejecuto correctamente los stage anteriores, disponemos de nuestra imagen de docker de nuestra app en ECR.
Podremos entonces desplegar esta imagen en **ECS** mediante el script _/service_up.sh_ el cual esta en este repositorio.
Si revisamos el stage **deploy-test** se repite el paso de instalar la CLI de AWS y la CLI de ECS para poder hacer el despliegue.

Debemos disponer de un file **.env** con nuestras variables necesarias para la configuracion segun el ambiente donde despleguemos y poder utilizarlas para el script. Como ejemplo usamos _.env.test_. Y como ya se menciono esta etapa solo ejecuta en la rama _develop_.

Debemos tener este archivo con los valores correspondientes a la configuracion del ambiente que querramos y luego hacer uso del archivo en el pipeline. Para produccion tendriamos _env.prod_ con distintos valores en las variables ya que no es el mismo entorno que test.

Nos enfocamos en el script: En este paso debemos de disponer de el docker-compose.yaml necesario para desplegar nuestra aplicacion en ECS.

Tenemos que inicializar el entonro ECS y para eso usaremos para gestionar ECS.
 _ecs-cli configure profile command_ 
 _ecs-cli configure --cluster_ De esta manera configuramos nuestro cluster como Fargate service.
 
 Con el comando _ecs-cli compose_ podremos desplegar nuestra app en el cluster definiendo la task en este paso, ademas este repositorio cuenta con el docker-compose.yml necesario para el despligue de la tarea, obteniendo la imagen de docker de la app del repositorio ECR visto en los pasos anteriores.
 Dentro de este docker-compose tambien definimos los logs de la app en AWS:
 
 ```yml
     logging:
      driver: awslogs
      options:
        awslogs-group: ${AWS_FRONT_LOGS_GROUP}
        awslogs-region: ${AWS_REGION}
        awslogs-stream-prefix: front

```
Tambien se debe crear un nuevo archivo llamado _ecs-params .yml_ que contiene las configuraciones de su ECS Cluster y ECS Service. En este archivo, puede especificar: 

- La configuración de red con su vpc y subredes. 
- Configuración de tareas: propiedades como límites de CPU y RAM para implementar el servicio. 

Todos estos parametros suponemos tener los datos ya que todos los recursos para realizar el deployment deben estar ya creados mediante Terraform

- VPC
- Sub-net
- Grupos de Seguridad
- Load Balancer
- Cluster
- Espacio de Nombres para Service Discovery
- Roles IAM


```yml
version: 1
task_definition:
  task_execution_role: YOUR_ECS_TASK_EXECUTION_ROLE_NAME
  ecs_network_mode: awsvpc
  task_size:
    mem_limit: 0.5GB
    cpu_limit: 256
run_params:
  network_configuration:
    awsvpc_configuration:
      subnets:
        - "YOUR SUBNET ID 1"
        - "YOUR SUBNET ID 2"
      security_groups:
        - "YOUR SECURITY GROUP ID"
      assign_public_ip: ENABLED
```      
Este reposiorio tiene un ejemplo un file _ecs-params.yml_ en el que de definen los parametros necesarios para esta tarea. Es decir CPU, MEM necesaria, como tambien el networking de redes o subredes segun se haya planificado la infraestructura.

```bash
#/bin/bash

# Prerequisitos
# -------------
# Deben estar establecidas las variables de entorno en el pipeline como el archivo .env
# según se indica en README.md

# Inicializacion de entorno para ECS
ecs-cli configure profile --profile-name ${ECS_ENVIRONMENT}-${APPLICATION}
ecs-cli configure --cluster ${ECS_ENVIRONMENT}-${APPLICATION} --default-launch-type FARGATE --config-name ${ECS_ENVIRONMENT}-${APPLICATION} --region ${AWS_REGION}

###########################
# Servicio app-front
ecs-cli compose --project-name ${ECS_ENVIRONMENT}-app-front \
    --cluster-config ${ECS_ENVIRONMENT}-${APPLICATION} \
    --ecs-profile ${ECS_ENVIRONMENT}-${APPLICATION} \
    --file ./front/docker-compose.yml \
    --ecs-params ./front/ecs-params.yml \
      service up \
    --enable-service-discovery \
    --target-groups "${AWS_FRONT_LB},containerName=front,containerPort=3000" \
    --tags Environment=${ECS_ENVIRONMENT},Application=${APPLICATION},CI_CD=true

# Scale app-front
ecs-cli compose --project-name ${ECS_ENVIRONMENT}-app-front \
    --cluster-config ${ECS_ENVIRONMENT}-${APPLICATION} \
    --ecs-profile ${ECS_ENVIRONMENT}-${APPLICATION} \
    --file ./front/docker-compose.yml \
    --ecs-params ./front/ecs-params.yml \
      service scale ${ECS_FRONT_SCALE} \
    --deployment-max-percent ${ECS_FRONT_MAX_PERCENT} \
    --deployment-min-healthy-percent ${ECS_FRONT_MIN_PERCENT}
```

De manera similar se ejecuta el stage **deploy-prod** teniendo en cuenta el file **env** para este entorno y lograr la configuracon deseada. Y ademas ejecutando de manera manual el pipeline indicando la rama main y la variable APP_VERSION necesaria para poder ejecutarlo correctamente.