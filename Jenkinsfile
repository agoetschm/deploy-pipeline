pipeline {
    agent none
    

    stages {
        stage('Package chart') {
            agent {
                docker { 
                    image 'alpine/helm:3.7.2' 
                    args '-it --entrypoint=' // hack to keep the container running
                }
            }
            steps {
                echo 'Packaging...'
                sh 'helm package charts/ingest -d chart_out'
            }
        }
        stage('Publish chart') {
            agent any
            steps {
                echo 'Publishing...'
                sh '''
                CHART_FILE=$(find chart_out -name "*.tgz")
                curl -u admin:password http://nexus:8081/repository/helm-hosted/ --upload-file ${CHART_FILE} -v
                '''
            }
        }
    }
}