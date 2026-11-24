pipeline {
    agent any
    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Build Maven') {
            steps {
                dir('backend') {
                    sh 'bash ./mvnw compile'
                }
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -t saidsouha_4ssa15_gestiondesprojets:1.0 backend'
            }
        }
        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_TOKEN')]) {
                    sh '''
                        set +x
                        export DOCKER_CONFIG="$(mktemp -d)"
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT
                        image="saidsouha_4ssa15_gestiondesprojets:1.0"
                        remote_image="$DOCKERHUB_USER/$image"
                        printf '%s' "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USER" --password-stdin
                        docker tag "$image" "$remote_image"
                        docker push "$remote_image"
                    '''
                }
            }
        }
    }
}
