pipeline {
    agent { label 'node1' }
    tools {
        jdk 'jdk17'
        maven 'maven 3.9.6'  
    }

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
