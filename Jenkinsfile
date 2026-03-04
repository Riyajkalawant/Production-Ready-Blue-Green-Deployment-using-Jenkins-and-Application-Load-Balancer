pipeline {
    agent any

    environment {
        ACTIVE_TG = 'arn:aws:elasticloadbalancing:ap-south-1:996091555734:targetgroup/Blue-TG/b91caa4deb45b883'
        INACTIVE_TG = 'arn:aws:elasticloadbalancing:ap-south-1:996091555734:targetgroup/Green-TG/f30e2eec6f0a642b'
        ALB_LISTENER_ARN = 'arn:aws:elasticloadbalancing:ap-south-1:996091555734:listener/app/blue-green-ALB/b1cab7841348dd14/10e585a31de76566'
        GREEN_EC2_IP = '43.204.232.41'  // तुमची Green EC2 Public IP टाका
        BLUE_EC2_IP = '13.233.80.41'   // तुमची Blue EC2 Public IP टाका
        SSH_KEY_PATH = '/home/ubuntu/.ssh/jenkinsubuntu.pem'  // तुमची SSH Key Path टाका
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                echo "Code pulled from GitHub"
            }
        }

        stage('Deploy to Inactive Environment') {
            steps {
                script {
                    echo "Deploying to ${INACTIVE_TG}..."
                    sh "ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ubuntu@${GREEN_EC2_IP} 'sudo cp index.html /var/www/html/'"
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    echo "Performing Health Check..."
                    def healthCheck = sh(script: "curl -f http://${ALB_LISTENER_ARN}/index.html > /dev/null 2>&1", returnStatus: true)
                    if (healthCheck != 0) {
                        error "Health Check Failed! Rolling back..."
                    }
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                script {
                    echo "Switching Traffic to ${INACTIVE_TG}..."
                    sh "aws elbv2 modify-listener --listener-arn ${ALB_LISTENER_ARN} --default-actions 'Protocol=HTTP,Port=80,TargetGroupArn=${INACTIVE_TG}'"
                }
            }
        }
    }

    post {
        failure {
            echo "Deployment Failed! Rolling back..."
            sh "aws elbv2 modify-listener --listener-arn ${ALB_LISTENER_ARN} --default-actions 'Protocol=HTTP,Port=80,TargetGroupArn=${ACTIVE_TG}'"
        }
    }
}
