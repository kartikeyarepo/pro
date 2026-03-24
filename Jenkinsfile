pipeline {
    agent any

    stages {
        stage('GitClone') {
            steps {
                git 'https://github.com/kartikeyarepo/pro.git'
            }
        }
   stage('Validate') {
            steps {
                sh 'mvn validate'
            }
        }
    stage('compile') {
            steps {
                sh 'mvn compile'
            }
        }
    stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    stage('Package') {
            steps {
                sh 'mvn package'
            }
        }
        
    }
}
