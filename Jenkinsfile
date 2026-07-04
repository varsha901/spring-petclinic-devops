pipeline {
    agent any

    environment {
        ACR_NAME = 'varshapetclinicacr973'
        ACR_LOGIN_SERVER = 'varshapetclinicacr973.azurecr.io'
        IMAGE_NAME = 'spring-petclinic'
        RESOURCE_GROUP = 'petclinic'
        AKS_CLUSTER = 'aks-petclinic'
    }

    stages {
        stage('Clone') {
            steps {
                git branch: 'main', url: 'https://github.com/spring-projects/spring-petclinic.git'
            }
        }

        stage('Build') {
            steps {
                sh './mvnw clean compile'
            }
        }

        stage('Test') {
            steps {
                sh './mvnw test'
            }
        }

        stage('Package') {
            steps {
                sh './mvnw clean package'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh './mvnw sonar:sonar -Dsonar.projectKey=Spring-PetClinic'
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $ACR_LOGIN_SERVER/$IMAGE_NAME:$BUILD_NUMBER .'
                sh 'docker tag $ACR_LOGIN_SERVER/$IMAGE_NAME:$BUILD_NUMBER $ACR_LOGIN_SERVER/$IMAGE_NAME:latest'
            }
        }

        stage('Push to ACR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'acr-creds', usernameVariable: 'ACR_USER', passwordVariable: 'ACR_PASS')]) {
                    sh '''
                    echo $ACR_PASS | docker login $ACR_LOGIN_SERVER -u $ACR_USER --password-stdin
                    docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:$BUILD_NUMBER
                    docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                sh 'trivy image --exit-code 0 --severity LOW,MEDIUM $ACR_LOGIN_SERVER/$IMAGE_NAME:$BUILD_NUMBER'
                sh 'trivy image --exit-code 1 --severity HIGH,CRITICAL $ACR_LOGIN_SERVER/$IMAGE_NAME:$BUILD_NUMBER'
            }
        }

        stage('Deploy to AKS') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'azure-sp', usernameVariable: 'AZURE_CLIENT_ID', passwordVariable: 'AZURE_CLIENT_SECRET'),
                    string(credentialsId: 'azure-tenant-id', variable: 'AZURE_TENANT_ID')
                ]) {
                    sh '''
                    az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant $AZURE_TENANT_ID
                    az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER --overwrite-existing
                    kubectl set image deployment/spring-petclinic spring-petclinic=$ACR_LOGIN_SERVER/$IMAGE_NAME:$BUILD_NUMBER || true
                    kubectl apply -f k8s/db.yml
                    kubectl apply -f k8s/petclinic.yml
                    kubectl rollout status deployment/spring-petclinic
                    kubectl get pods
                    kubectl get svc
                    '''
                }
            }
        }
    }
}