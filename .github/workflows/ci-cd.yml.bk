name: build push and deploy image

on:
  push:
    branches: 
      - main   
    
  pull_request:
    branches: 
      - main

permissions:
    id-token: write
    contents: read

env:
  AWS_REGION: us-east-1
  AWS_ROLE: ${{ secrets.ACTION_ROLE }}
  ECR_REPO_NAME: dev
  IMAGE_TAG: ${{ github.run_number }}

jobs:
  build:
    runs-on: ubuntu-latest # github shared runner not self hosted
    steps:
      - name: Clone repo
        uses: actions/checkout@v3 # prebuild actions

      - name: Configure Aws credentials
        uses: aws-actions/configure-aws-credentials@v4
        with: 
          role-to-assume: ${{ env.AWS_ROLE }} #OIDC
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR  
        id: ecr-login  
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push Docker image
        id: build-and-push
        run: |
          IMAGE_URI=${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPO_NAME }}:${{ env.IMAGE_TAG }} 
          docker build -t $IMAGE_URI .
          docker push $IMAGE_URI

      - name: Set IMAGE_URI for later steps    
        run: echo "IMAGE_URI=${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPO_NAME }}:${{ env.IMAGE_TAG }}" >> $GITHUB_ENV

      - name: Scan Docker image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_URI }}  # Scans the built image for bugs
          format: "table"
          exit-code: "0"
          severity: "CRITICAL,HIGH"

      - name: Fill in the new image URI in the ECS task definition
        id: render-task-def
        run: |
          sed "s|<IMAGE_URI>|${{ env.IMAGE_URI }}|g" ${{ env.ECS_TASK_DEF }} > task-def-rendered.json

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: task-def-rendered.json
          service: ${{ env.ECS_SERVICE_NAME }}
          cluster: ${{ env.ECS_CLUSTER_NAME }}
          wait-for-service-stability: true
      


      

