pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-southeast-2'
        ECR_REGISTRY = credentials('ecr-registry-url')
        ECR_REPOSITORY = 'my-docker-repo'
        IMAGE_TAG = "${BUILD_NUMBER}"
        // EC2_HOST = credentials('ec2-host-dev')
        // ANSIBLE_INVENTORY = '/etc/ansible/hosts'
    }

    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    sh '''
                        docker build -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG} .
                        docker tag ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                    '''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
                        docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest
                    '''
                }
            }
        }

        stage('Deploy with Ansible') {
            steps {
                script {
                    sh '''
                        ansible-playbook -i -i inventory playbook.yml \
                            -e docker_image=${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG} \
                            -e region=${AWS_REGION} \
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}