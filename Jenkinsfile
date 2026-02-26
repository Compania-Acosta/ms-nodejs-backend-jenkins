pipeline {

    agent {
        docker { image 'devops-agent:latest' }
    }

    environment {
        APELLIDO = "acosta"
        ACR_NAME = "acrglobalcicd"
        ACR_LOGIN_SERVER = "${ACR_NAME}.azurecr.io"
        IMAGE_NAME = "my-nodejs-app-${APELLIDO}"
        RESOURCE_GROUP = "rg-cicd-terraform-app-baraujox"
        AKS_NAME = "aks-dev-eastus"
    }

    stages {

        // =========================
        // CI
        // =========================

        stage('[CI] Instalar dependencias') {
            steps {
                sh 'npm install'
            }
        }

        stage('[CI] Pruebas Unitarias') {
            steps {
                sh 'npm run test || true'
            }
        }

        stage('[CI] Pruebas Integración') {
            steps {
                sh 'npm run test:integration || true'
            }
        }

        stage('[CI] Azure Login') {
            steps {
                withCredentials([
                    string(credentialsId: 'azure-clientId',       variable: 'AZ_CLIENT_ID'),
                    string(credentialsId: 'azure-clientSecret',   variable: 'AZ_CLIENT_SECRET'),
                    string(credentialsId: 'azure-tenantId',       variable: 'AZ_TENANT_ID'),
                    string(credentialsId: 'azure-subscriptionId', variable: 'AZ_SUBSCRIPTION_ID')
                ]) {
                    sh '''
                      az login --service-principal \
                        --username="$AZ_CLIENT_ID" \
                        --password="$AZ_CLIENT_SECRET" \
                        --tenant="$AZ_TENANT_ID"

                      az account set --subscription $AZ_SUBSCRIPTION_ID
                    '''
                }
            }
        }

        stage('[CI] AKS Credentials') {
            steps {
                sh '''
                  az aks get-credentials \
                    --resource-group $RESOURCE_GROUP \
                    --name $AKS_NAME \
                    --overwrite-existing
                '''
            }
        }

        stage('[CI] Generar ID corto del commit') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    echo "IMAGE_TAG: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('[CI] Build and Push Docker Image') {
            steps {
                sh '''
                  az acr login --name $ACR_NAME

                  docker build -t $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG .
                  docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        // =========================
        // CONFIGURAR DEV
        // =========================

        stage('[CD-DEV] Configurar Variables') {
            steps {
                script {
                    env.ENV = "dev"
                    env.API_PROVIDER_URL = "https://dev.api.com"
                }
                sh '''
                  envsubst < k8s.yml > k8s-dev.yml
                '''
            }
        }

        stage('[CD-DEV] Deploy a AKS') {
            steps {
                sh 'kubectl apply -f k8s-dev.yml'
            }
        }

        stage('[CD-DEV] Imprimir IP del servicio') {
            steps {
                sh '''
                  SERVICE_NAME="my-nodejs-service-${APELLIDO}-dev"
                  kubectl get svc $SERVICE_NAME
                '''
            }
        }

        // =========================
        // QA
        // =========================

        stage('Aprobación QA') {
            steps {
                input message: "¿Aprobar despliegue a QA?"
            }
        }

        stage('[CD-QA] Configurar Variables') {
            steps {
                script {
                    env.ENV = "qa"
                    env.API_PROVIDER_URL = "https://qa.api.com"
                }
                sh '''
                  envsubst < k8s.yml > k8s-qa.yml
                '''
            }
        }

        stage('[CD-QA] Deploy a AKS') {
            steps {
                sh 'kubectl apply -f k8s-qa.yml'
            }
        }

        stage('[CD-QA] Imprimir IP del servicio') {
            steps {
                sh '''
                  SERVICE_NAME="my-nodejs-service-${APELLIDO}-qa"
                  kubectl get svc $SERVICE_NAME
                '''
            }
        }

        // =========================
        // PRD
        // =========================

        stage('Aprobación PRD') {
            steps {
                input message: "¿Aprobar despliegue a PRODUCCIÓN?"
            }
        }

        stage('[CD-PRD] Configurar Variables') {
            steps {
                script {
                    env.ENV = "prd"
                    env.API_PROVIDER_URL = "https://api.com"
                }
                sh '''
                  envsubst < k8s.yml > k8s-prd.yml
                '''
            }
        }

        stage('[CD-PRD] Deploy a AKS') {
            steps {
                sh 'kubectl apply -f k8s-prd.yml'
            }
        }

        stage('[CD-PRD] Imprimir IP del servicio') {
            steps {
                sh '''
                  SERVICE_NAME="my-nodejs-service-${APELLIDO}-prd"
                  kubectl get svc $SERVICE_NAME
                '''
            }
        }

    }
}