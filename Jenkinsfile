pipeline {
    agent any

    parameters {
        choice(name: 'TARGET_ENVIRONMENT', choices: ['blue', 'green'], description: 'Select the target environment')
    }

    stages {
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t your-spring-boot-image .'
            }
        }

        stage('Start Target Environment') {
            steps {
                script {
                    if (params.TARGET_ENVIRONMENT == 'green') {
                        sh 'docker-compose -f docker-compose-green.yml up -d'
                        sleep time: 10, unit: 'SECONDS'
                        sh 'curl http://localhost:8082 || exit 1'
                    } else if (params.TARGET_ENVIRONMENT == 'blue') {
                        sh 'docker-compose -f docker-compose-blue.yml up -d'
                        sleep time: 10, unit: 'SECONDS'
                        sh 'curl http://localhost:8081 || exit 1'
                    }
                }
            }
        }

        stage('Create/Update Nginx Config') {
            steps {
                script {
                    if (params.TARGET_ENVIRONMENT == 'green') {
                        sh '''
                            cat << EOF > /tmp/green.conf
                            server {
                                listen 80;
                                server_name yourdomain.com; # Replace with your domain or IP

                                location / {
                                    proxy_pass http://localhost:8082;
                                    proxy_set_header Host \$host;
                                    proxy_set_header X-Real-IP \$remote_addr;
                                    proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
                                }
                            }
                            EOF
                            sudo mv /tmp/green.conf /etc/nginx/sites-available/green
                        '''
                    } else if (params.TARGET_ENVIRONMENT == 'blue') {
                        sh '''
                            cat << EOF > /tmp/blue.conf
                            server {
                                listen 80;
                                server_name yourdomain.com; # Replace with your domain or IP

                                location / {
                                    proxy_pass http://localhost:8081;
                                    proxy_set_header Host \$host;
                                    proxy_set_header X-Real-IP \$remote_addr;
                                    proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
                                }
                            }
                            EOF
                            sudo mv /tmp/blue.conf /etc/nginx/sites-available/blue
                        '''
                    }
                }
            }
        }

        stage('Switch Traffic (Nginx)') {
            steps {
                script {
                    if (params.TARGET_ENVIRONMENT == 'green') {
                        sh 'sudo rm /etc/nginx/sites-enabled/blue || true'
                        sh 'sudo ln -s /etc/nginx/sites-available/green /etc/nginx/sites-enabled/'
                    } else if (params.TARGET_ENVIRONMENT == 'blue') {
                        sh 'sudo rm /etc/nginx/sites-enabled/green || true'
                        sh 'sudo ln -s /etc/nginx/sites-available/blue /etc/nginx/sites-enabled/'
                    }
                    sh 'sudo systemctl reload nginx'
                }
            }
        }

        stage('Verify Target Deployment') {
            steps {
                sh 'curl http://yourdomain.com || exit 1'
            }
        }

        stage('Stop Other Environment') {
            when { expression { currentBuild.result == 'SUCCESS' } }
            steps {
                script {
                    if (params.TARGET_ENVIRONMENT == 'green') {
                        sh 'docker-compose -f docker-compose-blue.yml down || true'
                    } else if (params.TARGET_ENVIRONMENT == 'blue') {
                        sh 'docker-compose -f docker-compose-green.yml down || true'
                    }
                }
            }
        }
        stage('Switch Blue to Green Configuration') {
            when { expression { currentBuild.result == 'SUCCESS' } }
            steps{
                script {
                    if (params.TARGET_ENVIRONMENT == 'green') {
                        sh '''
                        cp docker-compose-green.yml docker-compose-blue.yml
                        '''
                    } else if (params.TARGET_ENVIRONMENT == 'blue') {
                        sh '''
                        cp docker-compose-blue.yml docker-compose-green.yml
                        '''
                    }
                }
            }
        }
    }
    post {
        failure {
            echo 'Deployment failed'
        }
    }
}