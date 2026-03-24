pipeline {
    agent any

   stages {      
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
