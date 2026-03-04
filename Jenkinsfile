pipeline {
    agent any

    environment {
        ACTIVE_TG = 'arn:aws:elasticloadbalancing:ap-south-1:996091555734:targetgroup/Blue-TG/b91caa4deb45b883'
        INACTIVE_TG = 'arn:aws:elasticloadbalancing:ap-south-1:996091555734:targetgroup/Green-TG/f30e2eec6f0a642b'
        ALB_LISTENER_ARN = 'arn:aws:elasticloadbalancing:ap-south-1:996091555734:listener/app/blue-green-ALB/b1cab7841348dd14/10e585a31de76566'
        ALB_DNS = 'blue-green-ALB-372743362.ap-south-1.elb.amazonaws.com'
        GREEN_EC2_IP = '43.204.232.41'
        SSH_KEY_PATH = '/var/lib/jenkins/bluegreen.pem'
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
                echo "Code pulled from GitHub"
            }
        }

        stage('Deploy to Green Environment') {
            steps {
                script {
                    sh """
                    echo "Copying index.html to Green server..."

                    scp -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no index.html \
                    ec2-user@${GREEN_EC2_IP}:/home/ec2-user/

                    echo "Moving file to NGINX web directory..."

                    ssh -i ${SSH_KEY_PATH} -o StrictHostKeyChecking=no ec2-user@${GREEN_EC2_IP} '
                        sudo mv /home/ec2-user/index.html /usr/share/nginx/html/index.html
                        sudo systemctl restart nginx
                    '

                    echo "Deployment to Green completed."
                    """
                }
            }
        }

        stage('Wait for Green Registration') {
            steps {
                echo "Waiting 60 seconds for target group registration..."
                sleep 60
            }
        }

        stage('Verify Green Target Group Health') {
            steps {
                script {
                    echo "Checking Green Target Group Health..."

                    def status = sh(
                        script: """
                        aws elbv2 describe-target-health \
                        --target-group-arn ${INACTIVE_TG} \
                        --query 'TargetHealthDescriptions[*].TargetHealth.State' \
                        --output text
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Green TG Health Status: ${status}"

                    if (!status.contains("healthy")) {
                        error("Green Target Group is NOT healthy!")
                    }

                    echo "Green is Healthy!"
                }
            }
        }

        stage('Switch Traffic to Green') {
            steps {
                script {
                    echo "Switching traffic to Green Target Group..."

                    sh """
                    aws elbv2 modify-listener \
                    --listener-arn ${ALB_LISTENER_ARN} \
                    --default-actions Type=forward,TargetGroupArn=${INACTIVE_TG}
                    """

                    echo "Traffic switched to Green successfully!"
                }
            }
        }
    }

    post {

        failure {
            echo "Deployment Failed! Rolling back to Blue..."

            sh """
            aws elbv2 modify-listener \
            --listener-arn ${ALB_LISTENER_ARN} \
            --default-actions Type=forward,TargetGroupArn=${ACTIVE_TG}
            """

            echo "Rollback to Blue completed!"
        }

        success {
            echo "Blue-Green Deployment Successful "
        }
    }
}
