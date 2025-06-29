// ---> PASTE YOUR REAL ARNs HERE <---
def BLUE_TG_ARN = "arn:aws:elasticloadbalancing:ap-south-1:842517499452:targetgroup/blue-tg/abe593ed83cd16f2"
def GREEN_TG_ARN = "arn:aws:elasticloadbalancing:ap-south-1:842517499452:targetgroup/green-tg/2512444eccb6e20b"
def LISTENER_ARN = "arn:aws:elasticloadbalancing:ap-south-1:842517499452:listener/app/my-app-alb/b152e3cf6806334f/d5334709ce151d9d"

pipeline {
    agent any
    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
        AWS_DEFAULT_REGION    = 'ap-south-1' // Change if your region is different
    }
    stages {
        stage('1. Identify Inactive Environment') {
            steps {
                script {
                    def current_live_tg = sh(script: "aws elbv2 describe-listeners --listener-arns ${LISTENER_ARN} | jq -r '.Listeners[0].DefaultActions[0].TargetGroupArn'", returnStdout: true).trim()
                    if (current_live_tg == BLUE_TG_ARN) {
                        env.DEPLOY_TO_ENV = 'green'
                        env.DEPLOY_TO_TG_NAME = 'green-tg'
                    } else {
                        env.DEPLOY_TO_ENV = 'blue'
                        env.DEPLOY_TO_TG_NAME = 'blue-tg'
                    }
                    echo "Live traffic is NOT on ${env.DEPLOY_TO_ENV.toUpperCase()}. We will configure the ${env.DEPLOY_TO_ENV.toUpperCase()} server."
                }
            }
        }
        stage('2. Configure Inactive Server (Automated)') {
            steps {
                script {
                    echo "Running Ansible on the existing '${env.DEPLOY_TO_ENV}-server'..."
                    if (env.DEPLOY_TO_ENV == 'blue') {
                        ansiblePlaybook(playbook: 'configure_blue.yml')
                    } else {
                        ansiblePlaybook(playbook: 'configure_green.yml')
                    }
                }
            }
        }
        stage('3. Verify Inactive Server (Manual)') {
            steps {
                input "ACTION: The '${env.DEPLOY_TO_ENV}-server' is configured. In AWS, go to Target Groups -> '${env.DEPLOY_TO_TG_NAME}' and confirm the server's health status is 'healthy'. Click 'Proceed' when ready."
            }
        }
        stage('4. Switch Live Traffic (Manual)') {
            steps {
                input "FINAL ACTION: In AWS, go to Load Balancers -> your ALB -> Listeners. Edit the listener to forward traffic to '${env.DEPLOY_TO_TG_NAME}'. Test the ALB's DNS name in a browser. Click 'Proceed' when done."
            }
        }
        stage('5. Deployment Complete') {
            steps {
                echo "Success! The ${env.DEPLOY_TO_ENV.toUpperCase()} environment is now LIVE."
            }
        }
    }
}