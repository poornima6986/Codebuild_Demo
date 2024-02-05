# Coworking Space Service Extension
The Coworking Space Service is a set of APIs that enables users to request one-time tokens and administrators to authorize access to a coworking space. This service follows a microservice pattern and the APIs are split into distinct services that can be deployed and managed independently of one another.

For this project, you are a DevOps engineer who will be collaborating with a team that is building an API for business analysts. The API provides business analysts basic analytics data on user activity in the service. The application they provide you functions as expected locally and you are expected to help build a pipeline to deploy it in Kubernetes.

## Getting Started

### Deliverables
1. `Dockerfile`
2. Screenshot of AWS CodeBuild pipeline
3. Screenshot of AWS ECR repository for the application's repository
4. Screenshot of `kubectl get svc`
5. Screenshot of `kubectl get pods`
6. Screenshot of `kubectl describe svc <DATABASE_SERVICE_NAME>`
7. Screenshot of `kubectl describe deployment <SERVICE_NAME>`
8. All Kubernetes config files used for deployment (ie YAML files)
9. Screenshot of AWS CloudWatch logs for the application

### Dependencies
#### Local Environment
1. Python Environment - run Python 3.10+ applications and install Python dependencies via `pip`
2. Docker CLI - build and run Docker images locally
3. `kubectl` - run commands against a Kubernetes cluster
4. `helm` - apply Helm Charts to a Kubernetes cluster

#### Remote Resources
1. AWS CodeBuild - build Docker images remotely
2. AWS ECR - host Docker images
3. Kubernetes Environment with AWS EKS - run applications in k8s
4. AWS CloudWatch - monitor activity and logs in EKS
5. GitHub - pull and clone code

### Setup
The below sequence was followed to set up the project
1. Created a EKS Cluster named postgresql_EKS using the below commands
   eksctl create cluster --name postgresql-EKS --region us-east-1  --vpc-private-subnets subnet-0e8b6463ebfb4b527,subnet-0e9b27ca2bc123027 -P

2. Set up Bitnami Repo
helm repo add bitnami https://charts.bitnami.com/bitnami

2. Install PostgreSQL Helm Chart
helm install project3-eks bitnami/postgresql

This should set up a Postgre deployment at project3-eks-postgresql.default.svc.cluster.local 

By default, it will create a username `postgres`. The password can be retrieved with the following command:

export POSTGRES_PASSWORD=$(kubectl get secret --namespace default <SERVICE_NAME>-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

echo $POSTGRES_PASSWORD

<sup><sub>* The instructions are adapted from [Bitnami's PostgreSQL Helm Chart](https://artifacthub.io/packages/helm/bitnami/postgresql).</sub></sup>

3. Test Database Connection
The database is accessible within the cluster. This means that when you will have some issues connecting to it via your local environment. You can either connect to a pod that has access to the cluster _or_ connect remotely via [`Port Forwarding`](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)

* Connecting Via Port Forwarding
```bash
kubectl port-forward --namespace default svc/project3-eks-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432
```

* Connecting Via a Pod
```bash
kubectl exec -it project3-eks-postgresql-0 bash
PGPASSWORD="<PASSWORD HERE>" psql postgres://postgres@project3-eks-postgresql:5432/postgres -c <COMMAND_HERE>
```

4. Run Seed Files
We will need to run the seed files in `db/` in order to create the tables and populate them with data.

a.The below command will create the users and tokens tables and creates the required indices.

kubectl port-forward --namespace default svc/project3-eks-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < /workspace/db/1_create_tables.sql
b. The next step is to insert the users into the user table:
kubectl port-forward --namespace default svc/project3-eks-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < /workspace/db/2_seed_users.sql
c. the last step in the data insertion is to run the below command to insert the tokens into the tokens table:
kubectl port-forward --namespace default svc/project3-eks-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < /workspace/db/3_seed_tokens.sql

Deployment Steps:
1. Running CodeBuild:
 To use Code build, We can create a .yaml file (codebuild.yaml), which will help to create the repo and push the required images into the Code Repository:

 version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
      - cd analytics
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t analytics:latest .
      - docker tag analytics:latest $AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/analytics:latest
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/analytics:latest

2. Running the Analytics Application Locally
In the `analytics/` directory, we have got files to create the Deployment and the Services. The Config map and secret yaml files ocntains the Env details related to DB Username and DB Password.
Application can be created using the below commands (create the deployments and the services):

kubectl apply -f /workspace/analytics/deployment/deployment.yaml

kubectl apply -f /workspace/analytics/deployment/service.yaml

3. Verifying The Application
 export BASE_URL=$(kubectl get services analytics --output jsonpath='{.status.loadBalancer.ingress[0].hostname}')

* Generate report for check-ins grouped by dates
`curl $BASE_URL/api/reports/daily_usage`

* Generate report for check-ins grouped by users
`curl $BASE_URL/api/reports/user_visits`

Other Suggestions:
1. AWS instance that would best suit the application is m5 instacens, as they provide a balance of compute, memory, and network resources and the m5.large instance provides aroun 2vCPU and 8 GiB memory w hich is more than sufficient for the application.
2. We can also look to introduce HPA to save additional costs, which will automatically updates thw rokload to match the demand.