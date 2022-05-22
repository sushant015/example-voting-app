pipeline {
    agent {
      label 'worker'
    }

    options {
            buildDiscarder(logRotator(numToKeepStr: '10'))
            disableConcurrentBuilds()
            timeout(time: 1, unit: 'HOURS')
    }
    environment {
            AWS_DEFAULT_REGION = 'us-east-1'
            SERVICE_NAME = 'vote'
            TASK_FAMILY = 'vote-fargate'
            ECS_CLUSTER = 'vote-app'
    }
    stages {
      stage('Git Checkout') {
        steps {
          checkout scm
        }
      }
        stage('Build Docker Image') {
            steps {
                sh "eval \$(aws ecr get-login --no-include-email --region us-east-1) && sleep 2"
                sh "cd vote && sudo docker build . -t 052376543349.dkr.ecr.us-east-1.amazonaws.com/vote:\${BUILD_NUMBER}"
                sh "aws ecr get-login --no-include-email --region us-east-1"
                sh "aws ecr create-repository --repository-name 052376543349.dkr.ecr.us-east-1.amazonaws.com/vote --region us-east-1 || true"
                sh "sudo docker push 052376543349.dkr.ecr.us-east-1.amazonaws.com/vote:\${BUILD_NUMBER}"
            }
        }
        stage('Deploy in ECS') {
            steps {

                script {
                    sh'''
                    ECR_IMAGE="052376543349.dkr.ecr.us-east-1.amazonaws.com/vote:${BUILD_NUMBER}"
                    TASK_DEFINITION=$(aws ecs describe-task-definition --task-definition "$TASK_FAMILY" --region "$AWS_DEFAULT_REGION")
                    NEW_TASK_DEFINTIION=$(echo $TASK_DEFINITION | jq --arg IMAGE "$ECR_IMAGE" '.taskDefinition | .containerDefinitions[0].image = $IMAGE | del(.taskDefinitionArn) | del(.revision) | del(.status) | del(.requiresAttributes) | del(.compatibilities)')
                    NEW_TASK_INFO=$(aws ecs register-task-definition --region "$AWS_DEFAULT_REGION" --cli-input-json "$NEW_TASK_DEFINTIION")
                    NEW_REVISION=$(echo $NEW_TASK_INFO | jq '.taskDefinition.revision')
                    aws ecs update-service --cluster ${ECS_CLUSTER} \
                                        --service ${SERVICE_NAME} \
                                        --task-definition ${TASK_FAMILY}:${NEW_REVISION}'''
                }
            }
        }
    }

    post {
        always {
            deleteDir()
            sh "docker rmi 052376543349.dkr.ecr.us-east-1.amazonaws.com/vote:\${BUILD_NUMBER}"
            }
        }
}
