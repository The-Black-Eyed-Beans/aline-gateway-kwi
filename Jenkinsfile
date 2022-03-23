pipeline {
    agent any

    tools {
        maven "Maven"
    }

    environment {
        AWS_ID = credentials("AWS-ACCOUNT-ID")
        REGION = credentials("REGION-KWI")
        PROJECT = "gateway"
        COMMIT_HASH = "${sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()}"
    }

    stages {
        stage("Test") {
            steps {
                sh "git submodule init"
                sh "git submodule update"
                sh "mvn clean test"
            }
        }

        stage("Package")  {
            steps {
                sh "mvn package -DskipTests"
            }
        }

        stage("Docker Build") {
            steps {
                sh "aws ecr get-login-password --region ${REGION} --profile keshaun | docker login --username AWS --password-stdin ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com"
                sh "docker build -t ${PROJECT}-kwi:${COMMIT_HASH} ."
                sh "docker tag ${PROJECT}-kwi:${COMMIT_HASH} ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"
                sh "docker push ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"
            }
        }

        stage("Deployment") {
            steps {
                echo "Deploying ${PROJECT}-kwi..."
                sh '''
                aws cloudformation deploy \
                --stack-name ${PROJECT}-kwi-stack \
                --template-file gateway.json \
                --profile keshaun \
                --capabilities CAPABILITY_IAM \
                --no-fail-on-empty-changeset \
                --parameter-overrides \
                    MicroserviceName=${PROJECT} \
                    AppPort=80 \
                    ImageTag=${COMMIT_HASH}
                '''
            }
        }
    }

    post {
        always {
            sh "docker image rm ${PROJECT}-kwi:${COMMIT_HASH}"
            sh "docker image rm ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"
            sh "mvn clean"
        }
    }
}