pipeline {
    agent any

    environment {
        DOCKERHB_CREDENTIALS = credentials('dockerhub')
        GIT_REPO = 'https://github.com/dinhcam89/eks_cicd.git'
        MANIFEST_PATH = 'dist/kubernetes/'
        DEPLOYMENT_FILE = 'deploy.yaml'
        GIT_REPO_NAME = 'eks_cicd'
        GLOBAL_ENVIRONMENT = 'NO_DEPLOYMENT'
        ENVIRONMENT_STAGING = 'staging'
        VERSION = "${env.BUILD_NUMBER}"
        TAG = ''
    }

    stages {
        stage('Setup environment') {
            steps {
                echo 'Setting up environment'
                script {
                    // Determine whether this is staging or production build
                    switch (env.BRANCH_NAME) {
                        case 'dev':
                            GLOBAL_ENVIRONMENT = 'staging'
                            TAG = 'staging' + '-v1.' + VERSION
                            break
                        case 'main':
                            GLOBAL_ENVIRONMENT = 'prod'
                            TAG = 'prod' + '-v1.' + VERSION
                            break
                        default: 
                            GLOBAL_ENVIRONMENT = 'NO_DEPLOYMENT'
                            break
                    }
                }
                echo "Environment: ${GLOBAL_ENVIRONMENT}"
            }
        }
        // stage('SCA with OWASP Dependency Check') {
        //     steps {
        //         dependencyCheck additionalArguments: '''
        //             -o './'
        //             -s './'
        //             -f 'ALL'
        //             --prettyPrint''', odcInstallation: 'Dependency-Check'

        //         dependencyCheckPublisher pattern: 'dependency-check-report.xml'
        //     }
        // }
        // stage('SonarQube Analysis') {
        //     steps {
        //         script {
        //             // requires SonarQube Scanner 2.8+
        //             scannerHome = tool 'SonarScanner'
        //         }
        //         withSonarQubeEnv('SonarQube Server') {
        //             sh "${scannerHome}/bin/sonar-scanner \
        //             -Dsonar.projectKey=retail-shop-microservices \
        //             -Dsonar.java.binaries=."
        //         }
        //     }
        // }
        stage('Login to Docker Hub') {
            steps {
                sh 'sudo su - jenkins'
                sh 'echo $DOCKERHB_CREDENTIALS_PSW | echo $DOCKERHB_CREDENTIALS_USR | docker login -u $DOCKERHB_CREDENTIALS_USR -p $DOCKERHB_CREDENTIALS_PSW'
            }
        }
        stage('Build Docker Images') {
            steps {
                sh "chmod +x -R ${env.WORKSPACE}"
                sh "scripts/build-image.sh -s assets -t ${TAG}"
                sh "scripts/build-image.sh -s cart -t ${TAG}"
                sh "scripts/build-image.sh -s catalog -t ${TAG}"
                sh "scripts/build-image.sh -s checkout -t ${TAG}"
                sh "scripts/build-image.sh -s orders -t ${TAG}"
                sh "scripts/build-image.sh -s ui -t ${TAG}"
            }
        }
        stage('View Images') {
            steps {
                sh 'docker images'
            }
        }
        stage('Push Images to Docker Hub') {
            steps {
                sh "docker push dinhcam89/retail-store-catalog:${TAG}"
                sh "docker push dinhcam89/retail-store-cart:${TAG}"
                sh "docker push dinhcam89/retail-store-orders:${TAG}"
                sh "docker push dinhcam89/retail-store-checkout:${TAG}"
                sh "docker push dinhcam89/retail-store-assets:${TAG}"
                sh "docker push dinhcam89/retail-store-ui:${TAG}"
            }
        }
        stage('Update value in helm-chart') {
            steps {
				withCredentials([usernamePassword(credentialsId: 'github-credential', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
				sh """#!/bin/bash
					   git clone ${GIT_REPO} --branch ${env.BRANCH_NAME}
					   cd ${GIT_REPO_NAME}/${MANIFEST_PATH}
					   sed -i "s/{{IMAGE_TAG}}/${TAG}/g" ${DEPLOYMENT_FILE}
					   git checkout dev
                       git add . ; git commit -m "Update deployment file to version ${TAG}";git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/dinhcam89/eks_cicd.git
					   cd ..
					   [[ -d ${GIT_REPO_NAME} ]] && rm -r ${GIT_REPO_NAME}
					   """		
				}				
            }
        }
        // stage('Scan Docker Images with Trivy') {
        //     steps {
        //         sh 'TMPDIR=/home/jenkins'
        //         sh "trivy image --format template --template '@/usr/bin/html.tpl' -o trivy-report-catalog.html dinhcam89/retail-store-catalog:${TAG}"
        //         sh "trivy image --format template --template '@/usr/bin/html.tpl' -o trivy-report-cart.html dinhcam89/retail-store-cart:${TAG}"
        //         sh "trivy image --format template --template '@/usr/bin/html.tpl' -o trivy-report-orders.html dinhcam89/retail-store-orders:${TAG}"
        //         sh "trivy image --format template --template '@/usr/bin/html.tpl' -o trivy-report-checkout.html dinhcam89/retail-store-checkout:${TAG}"
        //         sh "trivy image --format template --template '@/usr/bin/html.tpl' -o trivy-report-assets.html dinhcam89/retail-store-assets:${TAG}"
        //         sh "trivy image --format template --template '@/usr/bin/html.tpl' -o trivy-report-ui.html dinhcam89/retail-store-ui:${TAG}"
        //     }
        // }
    }
    post {
        always {
            cleanWs()
            // sh 'docker rmi -f $(docker images -aq)'
            sh 'docker logout'
        //
        }
    }
}
