pipeline {
    agent any

    environment {
        ACTIVE_TG = 'arn:aws:elasticloadbalancing:ap-south-1:996091555734:targetgroup/Blue-TG/b91caa4deb45b883'
        INACTIVE_TG = 'arn:aws:elasticloadbalancing:ap-south-1:996091555734:targetgroup/Green-TG/f30e2eec6f0a642b'
        ALB_LISTENER_ARN = 'arn:aws:elasticloadbalancing:ap-south-1:996091555734:listener/app/blue-green-ALB/b1cab7841348dd14/10e585a31de76566'
        ALB_DNS = 'blue-green-ALB-372743362.ap-south-1.elb.amazonaws.com'   // 🔥 IMPORTANT
        GREEN_EC2_IP = '43.204.232.41'
        SSH_KEY_PATH = '/var/lib/jenkins/jenkinsubuntu.pem'
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
                    echo "Deploying to Green Server..."
                    sh """
                    ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no \
                    ubuntu@${GREEN_EC2_IP} 'sudo cp index.html /var/www/html/'
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    echo "Performing Health Check..."
                    def healthCheck = sh(
                        script: "curl -f http://${ALB_DNS}/index.html",
                        returnStatus: true
                    )
                    if (healthCheck != 0) {
                        error "Health Check Failed!"
                    }
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                script {
                    echo "Switching Traffic to Green..."
                    sh """
                    aws elbv2 modify-listener \
                    --listener-arn ${ALB_LISTENER_ARN} \
                    --default-actions Type=forward,TargetGroupArn=${INACTIVE_TG}
                    """
                }
            }
        }
    }

    post {
        failure {
            echo "Deployment Failed! Rolling back..."

            sh """
            aws elbv2 modify-listener \
            --listener-arn ${ALB_LISTENER_ARN} \
            --default-actions Type=forward,TargetGroupArn=${ACTIVE_TG}
            """
        }
    }
}
