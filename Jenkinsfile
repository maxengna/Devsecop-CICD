pipeline {
    agent any
    //test pipeline4
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'dev', url: 'https://github.com/maxengna/Devsecop-CICD.git'
            }
        }
        // stage('SonarQube Analysis') {
        //     steps {
        //         withSonarQubeEnv('sonar-server') {
        //             sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix -Dsonar.projectKey=Netflix'''
        //         }
        //     }
        // }
        // stage('Quality Gate') {
        //     steps {
        //         script {
        //             waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
        //         }
        //     }
        // }
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        // stage('OWASP FS Scan') {
        //     steps {
        //         dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
        //         dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        //     }
        // }
        stage('TRIVY FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }
        // test cicd pipeline2
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        withCredentials([string(credentialsId: 'tmdb-api-key', variable: 'TMDB_V3_API_KEY')]) {
                            sh '''
                    docker build \
                      --build-arg TMDB_V3_API_KEY=$TMDB_V3_API_KEY \
                      -t netflix .

                    docker tag netflix maxdev888/netflix:${BUILD_NUMBER}
                    docker push maxdev888/netflix:${BUILD_NUMBER}
                    '''
                        }
                    }
                }
            }
        }

        stage('TRIVY Image Scan') {
            steps {
                sh 'trivy image maxdev888/netflix:${BUILD_NUMBER} > trivyimage.txt'
            }
        }
        stage('Update Deployment YAML') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-creds', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh '''
                        git config user.email "phanupong.w2019@gmail.com"
                        git config user.name "maxengna"
                        git rev-parse --show-toplevel
                        cd $(git rev-parse --show-toplevel)

                        git checkout deploy || git checkout -b deploy
                        git fetch origin
                        git pull origin deploy --rebase || true

                        cd Kubernetes
                        sed -i 's#image: maxdev888/netflix:.*#image: maxdev888/netflix:'${BUILD_NUMBER}'#' deployment.yml

                        git status
                        git add -A
                        git commit -m "Update image version to ${BUILD_NUMBER}" || true
                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/maxengna/Devsecop-CICD.git deploy
                        '''
                    }
                }
            }
        }
        stage('Deploy to Container') {
            steps {
                sh '''
                docker ps -a --filter "name=netflix" -q | xargs -r docker rm -f
                docker run -d -p 8081:80 maxdev888/netflix:${BUILD_NUMBER}

                '''
            }
        }
    }
    post {
        always {
            emailext(
                attachLog: true,
                subject: "${currentBuild.result}",
                body: """Project: ${env.JOB_NAME}<br/>
                         Build Number: ${env.BUILD_NUMBER}<br/>
                         URL: ${env.BUILD_URL}<br/>""",
                to: 'phanupong.w2019@gmail.com',
                attachmentsPattern: 'trivyfs.txt, trivyimage.txt'
            )
        }
    }
}
