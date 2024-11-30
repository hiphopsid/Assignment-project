pipeline {
    agent any
        
    parameters{
        string(name: 'DOCKER_TAG', defaultValue: 'latest', description: "Docker tag")
    }
    
    tools {
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git checkout') {
            steps {
                git branch: 'main', credentialsId: '647eb1c2-54f3-4396-9021-5a80e446a291', url: 'https://github.com/hiphopsid/Multi-Tier-BankApp-CI.git'
            }
        }
        
         stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }
        
         stage('Unit test cases') {
            steps {
                sh 'mvn test -DskipTests=true'
            }
        }
        
         stage('Scan Dependencies') {
            steps {
                sh 'trivy fs --format table -o fs.html .'
            }
        }
        
        stage('OWASP Dependencies Check') {
            steps {
                dependencyCheck additionalArguments: '--format HTML', odcInstallation: 'DP-Check'
            }
        }
        
        stage('Sonar Check') {
            steps {
                withSonarQubeEnv('Sonarqube Server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=bankapp -Dsonar.projectKey=bankapp \
                        -Dsonar.java.binaries=target '''
                }
            }
        }
        
        stage('Build and Publish to nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings-devSecOps', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }
        
        stage('Docker Build and Tag') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker') {
                        sh '''
                        docker build -t hiphopsid/bankapp:${DOCKER_TAG} .
                        '''
                    }
                }
            }
        }
        
        stage('Docker Image scan') {
            steps {
                sh 'trivy image --format table -o dimage.html hiphopsid/bankapp:${DOCKER_TAG}'
            }
        }
        
        stage('Docker Push') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker push hiphopsid/bankapp:${DOCKER_TAG}"
                    }
                }
            }
        }
        
        stage('Update YAML Manifest in CD Repo') {
            steps {
                script{
                    withCredentials([gitUsernamePassword(credentialsId: '647eb1c2-54f3-4396-9021-5a80e446a291', gitToolName: 'Default')]) {
                            sh '''
                                rm -rf Multi-Tier-BankApp-CD
                                git clone https://github.com/hiphopsid/Multi-Tier-BankApp-CD.git
                                cd Multi-Tier-BankApp-CD
                                ls -l bankapp
                                repo_dir=$(pwd)
                                sed -i 's|image: hiphopsid/bankapp:.*|image: hiphopsid/bankapp:'${DOCKER_TAG}'|' ${repo_dir}/bankapp/bankapp-ds.yml
                            '''
                            
                            sh '''
                                echo "Updated YAML file contents:"
                                cat Multi-Tier-BankApp-CD/bankapp/bankapp-ds.yml
                            '''
                            
                            sh '''
                                cd Multi-Tier-BankApp-CD
                                git config user.email "siddhant.gupta_cs18@gla.ac.in"
                                git config user.name "hiphopsid"
                            '''
                            
                            sh '''
                                cd Multi-Tier-BankApp-CD
                                ls
                                git add bankapp/bankapp-ds.yml
                                git commit -m "update tags"
                                git push origin main
                            '''
                        }
                    }
                }
            }
        }
    }
