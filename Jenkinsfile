pipeline {
    agent none 

    stages {
        stage('Get env variables') { 
            agent any
            
            steps { 
                sh 'env' 
            }
        }
    }
}
