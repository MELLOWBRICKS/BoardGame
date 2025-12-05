pipeline {
    agent any
    
    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
    }
    
    stages {
        stage('Get EC2 Instance IPs') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    script {
                        // Get all running production instances
                        def instanceIPs = sh(
                            script: '''
                                aws ec2 describe-instances \
                                  --region ap-south-1 \
                                  --filters "Name=tag:Environment,Values=production" "Name=instance-state-name,Values=running" \
                                  --query 'Reservations[*].Instances[*].PublicIpAddress' \
                                  --output text | tr '\t' '\n'
                            ''',
                            returnStdout: true
                        ).trim()
                        
                        if (instanceIPs) {
                            env.INSTANCE_IPS = instanceIPs.split('\n').join(',')
                            echo "Found instances: ${env.INSTANCE_IPS}"
                        } else {
                            error("No running instances found!")
                        }
                    }
                }
            }
        }
        
        stage('Update Application - Zero Downtime') {
            steps {
                script {
                    def instances = env.INSTANCE_IPS.split(',')
                    
                    // Update each instance one by one
                    for (int i = 0; i < instances.length; i++) {
                        def instanceIP = instances[i]
                        
                        echo "Updating instance ${i + 1}/${instances.length}: ${instanceIP}"
                        
                        sshagent(['aws-deploy-key']) {
                            sh """
                                ssh -o StrictHostKeyChecking=no ubuntu@${instanceIP} '
                                    cd /home/ubuntu/BoardGame
                                    
                                    echo "Pulling latest code..."
                                    git pull origin main
                                    
                                    echo "Rebuilding Docker image..."
                                    docker compose build
                                    
                                    echo "Restarting container with zero downtime..."
                                    docker compose up -d --no-deps --build boardgame-app
                                    
                                    echo "Waiting for application to be ready..."
                                    sleep 30
                                    
                                    echo "Health check..."
                                    curl -f http://localhost:8080/login || exit 1
                                    
                                    echo "Instance ${instanceIP} updated successfully!"
                                '
                            """
                        }
                        
                        // Wait before updating next instance (if multiple)
                        if (i < instances.length - 1) {
                            echo "Waiting 20 seconds before updating next instance..."
                            sleep 20
                        }
                    }
                }
            }
        }
        
        stage('Verify All Instances') {
            steps {
                script {
                    def instances = env.INSTANCE_IPS.split(',')
                    
                    for (instanceIP in instances) {
                        sshagent(['aws-deploy-key']) {
                            sh """
                                ssh -o StrictHostKeyChecking=no ubuntu@${instanceIP} '
                                    docker ps | grep boardgame
                                    curl -f http://localhost:8080/login
                                '
                            """
                        }
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo '✅ Application updated successfully on all instances!'
            echo "Updated instances: ${env.INSTANCE_IPS}"
        }
        failure {
            echo '❌ Application update failed!'
            echo 'Check logs for details'
        }
    }
}
