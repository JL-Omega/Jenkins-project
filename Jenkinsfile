pipeline {

    agent any

    environment {
        IMAGE_NAME           = 'protem'
        IMAGE_TAG            = '1.0.0'
        DOCKER_IMAGE_NAME    = "${IMAGE_NAME}:${IMAGE_TAG}"
        DOCKER_HUB_USERNAME  = credentials('docker_hub_usernsme')
        DOCKER_HUB_PASSWORD  = credentials('docker_hub_password')
        EC2_PUBLIC_IP        = credentials('ec2_public_ip')
        EC2_USER             = credentials('ec2_user')
        EC2_PRIVATE_KEY      = credentials('ec2_private_key')
    }

    stages {
        stage("Build the image") {
            steps {
                sh "docker build -t ${DOCKER_IMAGE_NAME} ."
            }
        }

        stage("Test the image") {
            steps {
                sh """
                docker rm -f ${IMAGE_NAME} || true
                docker run --name ${IMAGE_NAME} -d -p 8085:80 ${DOCKER_IMAGE_NAME}
                sleep 10
                curl http://localhost:8085 | grep -i 'Protem'
                """
            }
        }
        
        stage("Release the image") {
            steps {
                sh """
                docker login -u ${DOCKER_HUB_USERNAME} -p ${DOCKER_HUB_PASSWORD}
                docker tag ${DOCKER_IMAGE_NAME} ${DOCKER_HUB_USERNAME}/${DOCKER_IMAGE_NAME}
                docker push ${DOCKER_HUB_USERNAME}/${DOCKER_IMAGE_NAME}
                 """
            }
        }

        stage("inslall docker on EC2") {
            when {
                    expression { GIT_BRANCH == 'origin/main'}
                }
            steps {
                sh """
                ssh -i ${EC2_PRIVATE_KEY} -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_PUBLIC_IP} curl -fsSL https://get.docker.com -o install-docker.sh
                ssh -i ${EC2_PRIVATE_KEY} -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_PUBLIC_IP} sh install-docker.sh --dry-run
                ssh -i ${EC2_PRIVATE_KEY} -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_PUBLIC_IP} sudo sh install-docker.sh
                ssh -i ${EC2_PRIVATE_KEY} -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_PUBLIC_IP} sudo apt-get install -y uidmap
                ssh -i ${EC2_PRIVATE_KEY} -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_PUBLIC_IP} dockerd-rootless-setuptool.sh install
                """
            }
        }

        stage("Deploy the app to EC2") {
            when {
                    expression { GIT_BRANCH == 'origin/main'}
                }
            steps {
                sh """
                ssh -i ${EC2_PRIVATE_KEY} -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_PUBLIC_IP} docker rm -f ${IMAGE_NAME} || true
                ssh -i ${EC2_PRIVATE_KEY} -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_PUBLIC_IP} docker run --name ${IMAGE_NAME} -d -p 8080:80 ${DOCKER_HUB_USERNAME}/${DOCKER_IMAGE_NAME}
                """
            }
        }
    }
}
