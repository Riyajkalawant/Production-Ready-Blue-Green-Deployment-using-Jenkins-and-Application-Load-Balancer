pipeline {
    agent any

    environment {
        LISTENER_ARN = "arn:aws:elasticloadbalancing:ap-south-1:996091555734:loadbalancer/app/blue-green-ALB/23e941fe53f8f456"
        BLUE_TG = "arn:aws:elasticloadbalancing:ap-south-1:996091555734:targetgroup/blue-tg/16a09727407136ae"
        GREEN_TG = "arn:aws:elasticloadbalancing:ap-south-1:996091555734:targetgroup/green-tg/1fe7388114450e41"
        BLUE_IP = "3.109.54.228"
        GREEN_IP = "13.203.232.193"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/Riyajkalawant/Production-Ready-Blue-Green-Deployment-using-Jenkins-and-Application-Load-Balancer.git'
            }
        }

        stage('Check Active Target Group') {
            steps {
                script {
                    ACTIVE_TG = sh(
                        script: "aws elbv2 describe-listeners --listener-arns $LISTENER_ARN --query 'Listeners[0].DefaultActions[0].TargetGroupArn' --output text",
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Deploy to Inactive Environment') {
            steps {
                script {
                    if (ACTIVE_TG == BLUE_TG) {
                        TARGET_IP = GREEN_IP
                        NEW_TG = GREEN_TG
                    } else {
                        TARGET_IP = BLUE_IP
                        NEW_TG = BLUE_TG
                    }

                    sh """
                    scp -o StrictHostKeyChecking=no index.html ec2-user@$TARGET_IP:/var/www/html/index.html
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    sleep 15
                    RESPONSE = sh(
                        script: "curl -s http://$TARGET_IP",
                        returnStdout: true
                    )

                    if (!RESPONSE.contains("Version")) {
                        error("Health Check Failed")
                    }
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                sh """
                aws elbv2 modify-listener \
                --listener-arn $LISTENER_ARN \
                --default-actions Type=forward,TargetGroupArn=$NEW_TG
                """
            }
        }
    }

    post {
        failure {
            echo "Deployment Failed - Rolling Back"
        }
    }
}
