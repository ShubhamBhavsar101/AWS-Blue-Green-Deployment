// ---> PASTE YOUR REAL ARNs HERE <---
def BLUE_TG_ARN = "arn:aws:elasticloadbalancing:ap-south-1:842517499452:targetgroup/blue-tg/abe593ed83cd16f2"
def GREEN_TG_ARN = "arn:aws:elasticloadbalancing:ap-south-1:842517499452:targetgroup/green-tg/2512444eccb6e20b"
def LISTENER_ARN = "arn:aws:elasticloadbalancing:ap-south-1:842517499452:listener/app/my-app-alb/b152e3cf6806334f/d5334709ce151d9d"

pipeline {
    agent any
    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
        AWS_DEFAULT_REGION    = 'ap-south-1'
    }
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/ShubhamBhavsar101/AWS-Blue-Green-Deployment'
            }
        }
        stage('Live Traffic Check') {
            steps {
                script {
                    def current_live_tg = sh(script: "aws elbv2 describe-listeners --listener-arns ${LISTENER_ARN} | jq -r '.Listeners[0].DefaultActions[0].TargetGroupArn'", returnStdout: true).trim()
                    def live_environment = (current_live_tg == BLUE_TG_ARN) ? "BLUE" : "GREEN"
                    echo "The live traffic is in ${live_environment}"
                    }
            }
        }
        stage('Confirmation to Deploy Code') {
            steps {
                script {
                    def current_live_tg = sh(script: "aws elbv2 describe-listeners --listener-arns ${LISTENER_ARN} | jq -r '.Listeners[0].DefaultActions[0].TargetGroupArn'", returnStdout: true).trim()

                    if (current_live_tg == BLUE_TG_ARN) {
                        env.DEPLOY_TO_ENV = 'green'
                        env.DEPLOY_TO_TG_ARN = GREEN_TG_ARN
                        env.SWITCH_TO_ARN = GREEN_TG_ARN
                    } else {
                        env.DEPLOY_TO_ENV = 'blue'
                        env.DEPLOY_TO_TG_ARN = BLUE_TG_ARN
                        env.SWITCH_TO_ARN = BLUE_TG_ARN
                    }
                    
                    input "The current live environment is NOT '${env.DEPLOY_TO_ENV.toUpperCase()}'. This pipeline will now configure and deploy to the '${env.DEPLOY_TO_ENV.toUpperCase()}' environment. Do you want to continue?"
                }
            }
        }

        stage('Configure Inactive Server (Automated)') {
            steps {
                script {
                    echo "Running Ansible on the existing '${env.DEPLOY_TO_ENV}-server'..."
                    // We MUST provide the inventory here so Ansible can find the remote EC2 hosts.
                    if (env.DEPLOY_TO_ENV == 'blue') {
                        ansiblePlaybook(
                            playbook: 'configure_blue.yml',
                            inventory: 'aws_ec2.yml'
                        )
                    } else {
                        ansiblePlaybook(
                            playbook: 'configure_green.yml',
                            inventory: 'aws_ec2.yml'
                        )
                    }
                }
            }
        }

        stage('Manual Approval to Go Live') {
            steps {
                script {
                    echo "Waiting 10 seconds for health checks to stabilize..."
                    sleep(10)
                    
                    def health = sh(
                        script: "aws elbv2 describe-target-health --target-group-arn ${env.DEPLOY_TO_TG_ARN} | jq -r '.TargetHealthDescriptions[0].TargetHealth.State'",
                        returnStdout: true
                    ).trim()
                    
                    if (health != 'healthy') {
                        error("Deployment failed: The new environment '${env.DEPLOY_TO_ENV}' is not healthy. Current status is '${health}'. Aborting.")
                    }
                    
                    echo "SUCCESS: The new '${env.DEPLOY_TO_ENV}' environment is configured and HEALTHY."
                    input "Ready to switch live traffic to the '${env.DEPLOY_TO_ENV}' environment?"
                }
            }
        }

        stage('Switch Live Traffic (Automated)') {
            steps {
                script {
                    echo "Switching live traffic automatically to ${env.DEPLOY_TO_ENV.toUpperCase()}..."
                    // The inventory parameter is NOT needed here because the playbook's
                    // 'hosts' is set to 'localhost', which Ansible always knows.
                    ansiblePlaybook(
                        playbook: 'traffic_switch.yml',
                        extraVars: [
                            listener_arn: LISTENER_ARN,
                            new_target_group_arn: env.SWITCH_TO_ARN
                        ]
                    )
                }
            }
        }
        
        stage('Deployment Complete') {
            steps {
                echo "Success! The ${env.DEPLOY_TO_ENV.toUpperCase()} environment is now LIVE."
            }
        }
    }
}