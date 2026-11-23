pipeline {
    agent { label 'jenkins-agent' }
    environment {
        REGISTRY = '192.168.20.90:5000'
        IMAGE1    = 'todo-app'
        IMAGE2    = 'todo-backend'
    }
    stages {
        stage('SonarQube analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('sonarcloud') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.host.url=\$SONAR_HOST_URL -Dsonar.token=\$SONAR_AUTH_TOKEN"
                    }
                }
            }
        }
        stage('Build image') {
            steps {
                sh 'docker build -f app/Dockerfile -t $REGISTRY/$IMAGE1:$BUILD_NUMBER app/'
                sh 'docker build -f backend/Dockerfile -t $REGISTRY/$IMAGE2:$BUILD_NUMBER backend/'
            }
        }
        stage('Push image') {
            steps {
                sh 'docker push $REGISTRY/$IMAGE1:$BUILD_NUMBER'
                sh 'docker push $REGISTRY/$IMAGE2:$BUILD_NUMBER'
            }
        }
        stage('Deploy') {
            steps {
                withCredentials([file(credentialsId: 'k3d-k3s', variable: 'KUBECONFIG')]) {
                    sh '''
                      sed -i "s|newTag:.*|newTag: \\"$BUILD_NUMBER\\"|" deployments/kustomization.yaml
                      kubectl --request-timeout=60s apply -k deployments/
                      kubectl -n project rollout status deployment/todo-app --timeout=180s
                      kubectl -n project rollout status deployment/todo-backend --timeout=180s
                    '''
                }
            }
        }
    }
}
