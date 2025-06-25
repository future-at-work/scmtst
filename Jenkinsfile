pipeline {
    agent any
    options {
        buildDiscarder(logRotator(numToKeepStr: '5', daysToKeepStr: '14'))
    }

    environment {
        GDN_MDC_CLI_TENANT_ID=credentials('TENANT_ID')
        GDN_MDC_CLI_CLIENT_ID=credentials('CLI_CLIENT_ID')
        GDN_MDC_CLI_CLIENT_SECRET=credentials('CLI_CLIENT_SECRET')
        GDN_PIPELINENAME="jenkins"
        GDN_TRIVY_ACTION="image"
        IMAGE_NAME="vulnerableimage"
        KUBECONFIG="/tmp/jenkins/.kube/config"
        PATH = "/tmp/jenkins/bin:$PATH"
        REPO = "https://gitlab.com/gitlab-org/project-templates/spring"
        CLEAN = 'false'
    }

    stages {  
        stage('Install Tools') {
            steps {
                script {
                    sh '''
                        mkdir -p /tmp/jenkins/bin
                        curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"
                        chmod +x ./kubectl
                        mv ./kubectl /tmp/jenkins/bin/kubectl
                    '''
                }
            }
        }

        stage('Clone') {
            steps {
                echo 'Cloning Repository...'
                git branch: 'main', url: 'https://gitlab.com/gitlab-org/project-templates/spring'
            }
        }

        stage('Build Docker Container') {
            steps {
                echo 'Building Docker Container...'
                withCredentials([string(credentialsId: 'ACR_URI', variable: 'DOCKER_REGISTRY'),
                                 string(credentialsId: 'ACR_USERNAME', variable: 'DOCKER_USERNAME'),
                                 string(credentialsId: 'ACR_PASSWORD', variable: 'DOCKER_PASSWORD')]) {
                    sh '''
                      GDN_TRIVY_TARGET="${DOCKER_REGISTRY}/${IMAGE_NAME}"
                      docker build -t $IMAGE_NAME .
                      docker login ${DOCKER_REGISTRY} --username ${DOCKER_USERNAME} --password ${DOCKER_PASSWORD}
                      docker tag ${IMAGE_NAME} ${GDN_TRIVY_TARGET}:V${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage ('Run Defender for Cloud CLI) {
            steps {
                script {
                        withCredentials([string(credentialsId: 'ACR_URI', variable: 'DOCKER_REGISTRY')]) {
                            
                            sh '''
                              GDN_TRIVY_TARGET="${DOCKER_REGISTRY}/${IMAGE_NAME}:V${BUILD_NUMBER}"
                              export GDN_TRIVY_TARGET
                              curl -L -o ./msdo_linux.zip "https://www.nuget.org/api/v2/package/Microsoft.Security.DevOps.Cli.linux-x64/"
                              unzip -o ./msdo_linux.zip
                              chmod +x tools/guardian
                              chmod +x tools/Microsoft.Guardian.Cli
                              tools/guardian init --force
                              tools/guardian run -t trivy --export-file ./ubuntu-test.sarif --publish-file-folder-path ./ubuntu-test.sarif --not-break-on-detections
                              docker push $GDN_TRIVY_TARGET
                            '''
                        }
                    }
                }
            }

        stage('Manual Approval') {
            input {
                message 'Approve to continue?'
            }
            steps {
                echo 'Deployment approved!'
            }
        }

        stage('Create Deployment YAML') {
            steps {
                script {
                  withCredentials([string(credentialsId: 'ACR_URI', variable: 'DOCKER_REGISTRY')]) {
                    def previousBuildNumber = (env.BUILD_NUMBER as Integer) - 1
                    GDN_TRIVY_TARGET="${DOCKER_REGISTRY}/${IMAGE_NAME}"
                    echo "Creating deployment for: ${GDN_TRIVY_TARGET}:V${env.BUILD_NUMBER}"
                    writeFile file: 'deployment.yaml', text: """
                        apiVersion: v1
                        kind: Pod
                        metadata:
                          name: gatedeployment
                          # namespace: mdclab 
                        spec:
                          containers:
                            - name: gateddeployment
                              image: ${GDN_TRIVY_TARGET}:V${env.BUILD_NUMBER}
                    """
                }
            }
        }
}
        stage('Deploy to AKS') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'KUBECONFIG', variable: 'KUBECONFIG')]) {
                        sh '''
                        kubectl apply -f deployment.yaml
                        '''
                    }
                }
            }
        }

        stage ('Clean Workspace') {
            steps {
                script {
                    sh 'ls'
                    cleanWs()
                    sh 'ls'
                }
            }
        }
    }
}
