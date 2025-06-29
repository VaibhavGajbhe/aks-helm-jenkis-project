pipeline {
    agent any

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Select target environment')
    }

    environment {
        ACR_NAME = 'youracrname.azurecr.io'
        IMAGE_NAME = 'your-image-name'
        AKS_CLUSTER_NAME = "aks-${params.ENVIRONMENT}"
        AKS_RESOURCE_GROUP = "rg-${params.ENVIRONMENT}"
        KUBECONFIG_CREDENTIALS_ID = "aks-kubeconfig-${params.ENVIRONMENT}"
        HELM_RELEASE_NAME = "my-app-${params.ENVIRONMENT}"
        HELM_CHART_DIR = "helm/my-app"
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout Code') {
            steps {
                git url: 'https://github.com/your-org/your-repo.git', branch: 'main'
            }
        }

        stage('Azure Login & Kubeconfig') {
            steps {
                withCredentials([azureServicePrincipal('azure-credentials')]) {
                    sh '''
                        az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant $AZURE_TENANT_ID
                        az aks get-credentials --name $AKS_CLUSTER_NAME --resource-group $AKS_RESOURCE_GROUP --overwrite-existing
                    '''
                }
            }
        }

        stage('Generate Image Tag') {
            steps {
                script {
                    def tag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.IMAGE_TAG = tag
                    env.FULL_IMAGE = "${ACR_NAME}/${IMAGE_NAME}:${tag}"
                }
            }
        }

        stage('Deploy using Helm') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIALS_ID}", variable: 'KUBECONFIG')]) {
                    script {
                        def deployStatus = sh(
                            script: """
                                helm upgrade --install $HELM_RELEASE_NAME $HELM_CHART_DIR \
                                    --set image.repository=${ACR_NAME}/${IMAGE_NAME} \
                                    --set image.tag=${IMAGE_TAG} \
                                    --namespace ${params.ENVIRONMENT} \
                                    --create-namespace
                            """,
                            returnStatus: true
                        )

                        if (deployStatus != 0) {
                            echo "⚠️ Helm deployment failed. Attempting rollback..."
                            sh "helm rollback $HELM_RELEASE_NAME || echo 'No rollback available'"
                            error("Deployment failed and rollback triggered.")
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful for environment: ${params.ENVIRONMENT}"
        }
        failure {
            echo "❌ Deployment failed in environment: ${params.ENVIRONMENT}"
        }
    }
}
