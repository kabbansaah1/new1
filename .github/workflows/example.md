name: Deploy to Amazon ECS

on:
  push:
    branches:
      - "main"

env:
  AWS_REGION: us-east-2
  ECR_REPOSITORY: my-first-ecr-repo
  CONTAINER_NAME: my-first-task
  # Define common environment-specific ECS variables
  DEV_ECS_CLUSTER: my-cluster-dev
  STAGING_ECS_CLUSTER: my-cluster-staging
  PROD_ECS_CLUSTER: my-cluster-prod
  DEV_ECS_SERVICE: my-service-dev
  STAGING_ECS_SERVICE: my-service-staging
  PROD_ECS_SERVICE: my-service-prod
  # Define the common ECS task definition location
  ECS_TASK_DEFINITION: .github/workflows/task-definition.json

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    environment: production

    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        role-to-assume: arn:aws:iam::893688252992:role/Github
        role-session-name: Github
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
      with:
        mask-password: "true"

    - name: Build, tag, and push image to Amazon ECR
      id: build-image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

    - name: Fill in the new image ID in the Amazon ECS task definition
      id: task-def
      uses: aws-actions/amazon-ecs-render-task-definition@v1
      with:
        task-definition: ${{ env.ECS_TASK_DEFINITION }}
        container-name: ${{ env.CONTAINER_NAME }}
        image: ${{ steps.build-image.outputs.image }}

    - name: Deploy to Development
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: ${{ steps.task-def.outputs.task-definition }}
        service: ${{ env.DEV_ECS_SERVICE }}
        cluster: ${{ env.DEV_ECS_CLUSTER }}
        wait-for-service-stability: true

  promote-to-staging:
    needs: deploy
    runs-on: ubuntu-latest
    environment: staging

    steps:
    - name: Promote to Staging
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: ${{ steps.task-def.outputs.task-definition }}
        service: ${{ env.STAGING_ECS_SERVICE }}
        cluster: ${{ env.STAGING_ECS_CLUSTER }}
        wait-for-service-stability: true

  promote-to-production:
    needs: promote-to-staging
    runs-on: ubuntu-latest
    environment: production

    steps:
    - name: Promote to Production
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: ${{ steps.task-def.outputs.task-definition }}
        service: ${{ env.PROD_ECS_SERVICE }}
        cluster: ${{ env.PROD_ECS_CLUSTER }}
        wait-for-service-stability: true
