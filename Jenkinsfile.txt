pipeline {
    agent any
    
    environment {
        REPO_URL = 'https://github.com/vipulsaw/Container-Orchestration-1.git'
        REPO_NAME = 'Container-Orchestration-1'
        HELM_CHART_DIR = 'mern-chart'
        NAMESPACE = 'mern'
        RELEASE_NAME = 'mern-app'
        EC2_INSTANCE_IP = '54.145.162.64'
    }
    
    stages {
        stage('Checkout Source Code') {
            steps {
                checkout([$class: 'GitSCM', 
                         branches: [[name: '*/main']], 
                         userRemoteConfigs: [[url: env.REPO_URL]]])
                echo "Repository cloned successfully"
            }
        }
        
        stage('Copy Repository to Target EC2') {
            steps {
                sshagent(credentials: ['vipul']) {
                    script {
                        // Create directory on remote if it doesn't exist
                        sh """
                            ssh -o StrictHostKeyChecking=no ubuntu@${env.EC2_INSTANCE_IP} \
                            'mkdir -p ~/${env.REPO_NAME}'
                        """
                        
                        // Copy files using rsync
                        sh """
                            rsync -avz -e "ssh -o StrictHostKeyChecking=no" \
                            --exclude='.git' \
                            --delete \
                            ./ ubuntu@${env.EC2_INSTANCE_IP}:~/${env.REPO_NAME}/
                        """
                        
                        echo "Files copied to EC2 instance successfully"
                    }
                }
            }
        }
        
        stage('Install/Upgrade Helm Release') {
            steps {
                sshagent(credentials: ['vipul']) {
                    script {
                        // Run helm upgrade
                        sh """
                            ssh -o StrictHostKeyChecking=no ubuntu@${env.EC2_INSTANCE_IP} \
                            'cd ~/${env.REPO_NAME}/${env.HELM_CHART_DIR} && \
                            helm upgrade --install ${env.RELEASE_NAME} . \
                            --namespace ${env.NAMESPACE} \
                            --create-namespace'
                        """
                        
                        echo "Helm release upgraded successfully"
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sshagent(credentials: ['vipul']) {
                    script {
                        sh """
                            ssh -o StrictHostKeyChecking=no ubuntu@${env.EC2_INSTANCE_IP} \
                            'helm list -n ${env.NAMESPACE} && \
                            kubectl get pods -n ${env.NAMESPACE}'
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed! Check the logs for details.'
        }
    }
}
