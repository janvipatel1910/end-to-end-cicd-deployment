pipeline {
    agent any

    environment {
        AWS_REGION = 'eu-north-1'
        ECR_REPOSITORY = 'end-to-end-cicd-app'
        APP_SERVER = '172.31.42.232'
    }

    stages {
        stage('Checkout Verification') {
            steps {
                echo '===== SOURCE CODE RECEIVED ====='
                sh '''
                    echo "Build Number: ${BUILD_NUMBER}"
                    echo "Workspace: ${WORKSPACE}"
                    ls -la
                '''
            }
        }

        stage('Automated Tests') {
            steps {
                echo '===== RUNNING AUTOMATED TESTS ====='
                sh '''
                    python3 -m venv .venv
                    .venv/bin/pip install -r requirements.txt
                    .venv/bin/python -m pytest -v
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                echo '===== BUILDING DOCKER IMAGE ====='
                sh '''
                    docker build -t ${ECR_REPOSITORY}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Push Image to Amazon ECR') {
            steps {
                echo '===== PUSHING IMAGE TO AMAZON ECR ====='
                sh '''
                    AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
                    ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${BUILD_NUMBER}"

                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    docker tag ${ECR_REPOSITORY}:${BUILD_NUMBER} ${ECR_IMAGE}
                    docker push ${ECR_IMAGE}

                    echo "Image pushed successfully: ${ECR_REPOSITORY}:${BUILD_NUMBER}"
                '''
            }
        }

        stage('Deploy to Application EC2') {
            steps {
                echo '===== DEPLOYING TO APPLICATION EC2 ====='

                withCredentials([sshUserPrivateKey(
                    credentialsId: 'cicd-app-server-ssh',
                    keyFileVariable: 'SSH_KEY',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh '''
                        AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
                        ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                        ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${BUILD_NUMBER}"

                        ssh -o StrictHostKeyChecking=no \
                            -i "$SSH_KEY" \
                            "$SSH_USER@$APP_SERVER" \
                            "AWS_REGION='$AWS_REGION' ECR_REGISTRY='$ECR_REGISTRY' ECR_IMAGE='$ECR_IMAGE' bash -s" <<'REMOTE'

set -e

echo "===== LOGGING IN TO AMAZON ECR ====="
aws ecr get-login-password --region "$AWS_REGION" | \
    sudo docker login \
    --username AWS \
    --password-stdin "$ECR_REGISTRY"

echo "===== PULLING NEW IMAGE ====="
sudo docker pull "$ECR_IMAGE"

echo "===== REPLACING APPLICATION CONTAINER ====="
sudo docker rm -f cicd-app 2>/dev/null || true

sudo docker run -d \
    --name cicd-app \
    --restart unless-stopped \
    -p 80:5000 \
    "$ECR_IMAGE"

echo "===== CHECKING APPLICATION HEALTH ====="
sleep 5
curl --fail --silent --show-error http://localhost/health

REMOTE
                    '''
                }
            }
        }
    }
}
