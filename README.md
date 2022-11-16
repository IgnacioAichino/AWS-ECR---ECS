# Pipeline de GitLab para desplegar en ECS en AWS

## _GitFlow_

La idea principal detrás de GitFlow es aislar el trabajo en diferentes tipos de branches, lo que le permite adaptarse muy bien al proceso colaborativo que necesita un equipo de desarrollo. GitFlow está basado principalmente en tres branches que tienen una vida infinita:

**master:** contiene el código de producción.
**develop:** contiene el código que ha finalizado desarrollo.
**feature:** se crea a partir de develop cuando una nueva funcionalidad necesita ser desarrollada. Al finalizar el desarrollo se hace merge a develop nuevamente.

## Preparacion y fundamentos

Si establecemos una estrategia de ramas. Desarrollaremos el pipeline para lograr los deployments en ECS.
Este proyecto de ejemplo se utiliza Docker para automatizar el despliegue de la aplicacion.
Debemos contar con una cuenta de AWS en la cual crearemos un repositorio de imagenes de docker gracias al servicio de **ECR**

Este repositorio cuenta con un archivo **.yaml** el cual tiene desarrollado el pipelina para contruir una imagen de docker de nuestra app y desplegarla en un ambiente de test y produccion en AWS.

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

Tanto **prepare, build, publish, y deploy-test** deben ser automatizados sobre la rama develop de nuestro proyecto. En esta rama se encuentra las nuevas funcionalidades del codigo en desarrollo por lo que los commits realizados sobre esta raman desencadenan en la ejecucion de estos stage para preparar el entorno de contruccion de la imagen, conectarnos a ECR donde alojaremos la imagen creada de nuestra app y luegos desplegemos en un ambiente de test.
Se muestran mas adelante las tareas de CI/CD en cada etapa.

La etapa de **deploy-prod** puede ser manual o automatica segun la estrategia que decidamos. En este caso sera manual la ejecucion del pipeline esperando como argumento de entrada un valor de version de nuestra app a ser desplegada en un ambiente productivo.


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

Nos enfocamos en el script:
