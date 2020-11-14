# Trambo-CircleCI
Trambo, Marzo 26 2020

## Archivo .config/config.yml
This file contains the following blocks
1. version
Indicates the version of the config file
2. jobs
There is where we define all the task that will be executed after a commit
3. workflows
controll the execution flow of the jobs.

## jobs

### prebuild
the following steps are needed in order to update a ECS service with
new container.

#### Steps
 - Login ECR
 This command uses the aws acces id and aws secre access id to acces to ECR
 
  ```
   - run:
          name: "Login ECR"
          command: |
            $(aws ecr get-login --no-include-email --region us-west-2)
      
  ```
 - Build Image
 This docker command build a image from a Docker file stored in Image folder in the same repo
 
  ```
   - run:
          name: "Build Image"
          command: |
            cd Imagen
            docker build -t nginx-elmer-trambo:${CIRCLE_SHA1} .
      
  ```

 - Tag and push Image to repo

 This commands are used to upload the local image previously created to ECR.
 
  ```
 - run:
          name: "Tag and push Image to repo"
          command: |
            docker tag nginx-elmer-trambo:${CIRCLE_SHA1} $Uri_ECR:${CIRCLE_SHA1}
            docker push $Uri_ECR:${CIRCLE_SHA1}
  ```

 - Create task definition and register in specific service
 
 To create the new task definitio we use the cli comman aws ecs to create and register in a specific services
 
  ```
- run:
          name: "Create task definition and register in specific service"
          command: |
            aws ecs register-task-definition --family nuevaTask --container-definitions "[{\"name\":\"Nginx-Service\",\"portMappings\":[{\"hostPort\":80,\"protocol\":\"tcp\",\"containerPort\":80}],\"image\":\"$Uri_ECR:${CIRCLE_SHA1}\",\"cpu\":10,\"memory\":300,\"essential\":true}]"
  ```
   - Update service and attack new task definition

 This command trigger the proces of updating the ECS service
 
  ```
 - run:
          name: "Update service and attack new task definition"
          command: |
            aws ecs update-service --cluster $ClusterName --service $ServiceName --task-definition nuevaTask 
    
  ```
   - Wait until service reach steady state

 This command waits until the ECS cluster reaches a steable state
 
  ```
  - run:
          name: "Wait until service reach steady state"
          command: |
            aws ecs wait services-stable --cluster $ClusterName --services $ServiceName
    
  ```
## workflows
Here we indicate the jobs belogns to the workflow. If whe just list the jobs them will be excuted simultaniously, in this case we defina that build jobs requires that prebuild job have to be completed to start.
```

workflows:
  version: 2
  build_and_test:
    jobs:
      - prebuild
      - build:
          requires:
            - prebuild

```


