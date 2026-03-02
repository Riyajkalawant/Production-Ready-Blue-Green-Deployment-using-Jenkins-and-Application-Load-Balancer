pipeline {
    agent any

    stages {
        stage('Pull Code') {
            steps {
                git 'https://github.com/Riyajkalawant/Production-Ready-Blue-Green-Deployment-using-Jenkins-and-Application-Load-Balancer'
            }
        }

        stage('Build') {
            steps {
                echo "Deploying Blue-Green Application"
            }
        }
    }
}
