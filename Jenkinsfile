pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        AWS_REGION = 'ap-southeast-2'
        ECR_REGISTRY = credentials('ecr-registry-url')
        ECR_REPOSITORY = 'my-docker-repo'
        IMAGE_TAG = "${BUILD_NUMBER}"
        ANSIBLE_IMAGE   = 'cytopia/ansible:latest'
        // EC2_HOST = credentials('ec2-host-dev')
        // ANSIBLE_INVENTORY = '/etc/ansible/hosts'
    }

    stages {
        stage ('Checkout') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], 
                userRemoteConfigs: [[url: 'https://github.com/ronnie-cunanan/docker-jenkins-ansible.git']])
            }
        }

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
                     sh """
                        docker run --rm \
                          -v /home/ubuntu:/var/jenkins_home \
                          -v /home/ubuntu/.ssh:/root/.ssh \
                          ${ANSIBLE_IMAGE} \
                          ansible-playbook \
                            -i /var/jenkins_home/workspace/${JOB_NAME}/inventory \
                            /var/jenkins_home/workspace/${JOB_NAME}/playbook.yml \
                            -e docker_image=${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG} \
                            -e region=${AWS_REGION} \
                            --ssh-extra-args='-o StrictHostKeyChecking=no'
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful: ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
        }
        failure {
            echo "Build or deployment failed. Check stage logs."
        }
    }
}