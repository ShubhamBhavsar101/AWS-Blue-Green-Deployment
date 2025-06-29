# AWS-Blue-Green-Deployment

We will proceed in four major phases:
*   **Phase 1:** One-Time AWS Infrastructure Setup
*   **Phase 2:** One-Time Jenkins & IAM Security Setup
*   **Phase 3:** Preparing Your Automation Code
*   **Phase 4:** Running Your First Deployment

---

### **Phase 1: One-Time AWS Infrastructure Setup**

**Goal:** To build your entire static environment in the `ap-south-1` (Mumbai) region. You only perform these steps once.

#### **Step 1.1: Create Security Groups (Your Virtual Firewalls)**

1.  Navigate to the **EC2** service in your AWS Console (region `ap-south-1`).
2.  Go to `Network & Security` -> `Security Groups`.
3.  **Create the ALB's Firewall:**
    *   **Name:** `alb-sg`
    *   **Inbound rules:** Add a rule for `HTTP` with Source `Anywhere-IPv4` (`0.0.0.0/0`).
4.  **Create the Servers' Firewall:**
    *   **Name:** `app-sg`
    *   **Inbound rules:**
        *   **Rule 1 (for ALB):** Type `HTTP`, Source: your `alb-sg` security group.
        *   **Rule 2 (for Ansible/SSH):** Type `SSH`, Source: your Jenkins server's public IP address (e.g., `1.2.3.4/32`).

#### **Step 1.2: Create and Tag EC2 Instances**

1.  Go to **EC2 -> Instances -> Launch instances**.
2.  **Launch the BLUE Server:**
    *   **Name:** `blue-server`
    *   **OS:** Select **Amazon Linux** (e.g., Amazon Linux 2023).
    *   **Key pair name:** `VenomPC`.
    *   **Firewall (security groups):** Choose `Select existing security group` and pick **`app-sg`**.
    *   **Tags:** Add a tag with **Key:** `Deployment` and **Value:** `Blue`.
3.  **Launch the GREEN Server:**
    *   Repeat the process. **Name:** `green-server`. Use the same settings but for **Tags**, use **Key:** `Deployment` and **Value:** `Green`.

#### **Step 1.3: Create Target Groups & Register Instances**

1.  Go to **EC2 -> Target Groups -> Create target group**.
2.  **Create BLUE Group:** Name it `blue-tg` (`Instances` type). On the next page, register your `blue-server` to it.
3.  **Create GREEN Group:** Name it `green-tg`. Register your `green-server` to it.

#### **Step 1.4: Create and Configure the Application Load Balancer**

1.  Go to **EC2 -> Load Balancers -> Create Load Balancer**.
2.  Choose **Application Load Balancer**.
3.  **Configure:**
    *   **Name:** `my-app-alb`.
    *   **Security groups:** Select your **`alb-sg`**.
    *   **Listeners and routing:** Set the default action for the `HTTP:80` listener to forward to **`blue-tg`**.
4.  **Add "Dummy" Rules for Health Checks (Crucial Step):**
    *   After the ALB is created, select it, go to the **Listeners** tab, and click `Manage rules` for the HTTP:80 listener.
    *   **Add Rule 1:** IF `Path` is `/health-check-for-blue*`, THEN Forward to `blue-tg`.
    *   **Add Rule 2:** IF `Path` is `/health-check-for-green*`, THEN Forward to `green-tg`.
    *   Save the rules. This ensures both target groups are always health-checked.

---

### **Phase 2: One-Time Jenkins & IAM Security Setup**

**Goal:** To securely configure Jenkins to interact with AWS and your EC2 instances.

#### **Step 2.1: Create an IAM User for Jenkins**

1.  Go to the **IAM** service in AWS.
2.  Create a new user named `jenkins-user`.
3.  Attach the **`AdministratorAccess`** policy directly (for learning purposes; use more restrictive policies in production).
4.  Create an **access key** for this user (choose CLI as the use case).
5.  **Copy the Access Key ID and Secret Access Key** immediately.

#### **Step 2.2: Store AWS Keys in Jenkins Credentials**

1.  In Jenkins, go to **Manage Jenkins -> Credentials -> (global) -> Add Credentials**.
2.  **Add the Secret Key:** Kind: `Secret text`, Secret: *(paste secret key)*, ID: `aws-secret-key`.
3.  **Add the Access Key ID:** Kind: `Secret text`, Secret: *(paste access key ID)*, ID: `aws-access-key`.

#### **Step 2.3: Install Jenkins Server Dependencies**

Log in to your Jenkins server via SSH and run these commands to install all required software.

```bash
# Update package manager (example for Debian/Ubuntu)
sudo apt-get update -y

# Install tools
sudo apt-get install -y ansible python3-pip jq

# Install AWS Python SDKs using the system package manager (to avoid PEP 668 error)
sudo apt-get install -y python3-boto3

# Install the Ansible AWS Collection globally
sudo ansible-galaxy collection install amazon.aws
```

#### **Step 2.4: Install and Configure Jenkins Plugins**

1.  In **Manage Jenkins -> Plugins**, install the `Ansible Plugin`.
2.  In **Manage Jenkins -> Tools**, scroll to the **Ansible** section.
    *   Click **Add Ansible**.
    *   **Name:** `Default-Ansible`.
    *   **Path:** Enter the path to the Ansible installation directory (e.g., `/usr/bin`).

---

### **Phase 3: Preparing Your Automation Code**

**Goal:** Create all the necessary files in your Git repository.

#### **Step 3.1: Place the SSH Private Key on the Jenkins Server**

1.  Copy your `VenomPC.pem` file to your Jenkins server.
2.  Run these commands to move it to the correct location and set secure permissions:

    ```bash
    sudo mv /path/to/VenomPC.pem /var/lib/jenkins/VenomPC.pem
    sudo chown jenkins:jenkins /var/lib/jenkins/VenomPC.pem
    sudo chmod 400 /var/lib/jenkins/VenomPC.pem
    ```

#### **Step 3.2: Create Your Project Files**

In your Git repository (`AWS-Blue-Green-Deployment`), ensure you have the following files:

1.  **`group_vars/all.yml`** (Defines SSH connection details)
    ```yaml
    ansible_user: ec2-user
    ansible_ssh_private_key_file: /var/lib/jenkins/VenomPC.pem
    ansible_ssh_common_args: '-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
    ```

2.  **`aws_ec2.yml`** (The dynamic inventory file)
    ```yaml
    plugin: amazon.aws.aws_ec2
    regions:
      - ap-south-1
    keyed_groups:
      - key: tags
        prefix: tag
    ```

3.  **`configure_blue.yml`**
    ```yaml
    - name: Configure the BLUE Server
      hosts: tag_Deployment_Blue
      become: yes
      tasks:
        - name: Install httpd
          dnf: { name: httpd, state: present }
        - name: Copy index file
          copy: { src: files/index_blue.html, dest: /var/www/html/index.html }
        - name: Start httpd
          service: { name: httpd, state: started, enabled: yes }
    ```

4.  **`configure_green.yml`** (Identical to blue, but uses `index_green.html` and targets `tag_Deployment_Green`)

5.  **`traffic_switch.yml`**
    ```yaml
    - name: Switch ALB Live Traffic
      hosts: localhost
      connection: local
      gather_facts: no
      tasks:
        - name: Modify ALB listener to point to the new Target Group
          command: >
            aws elbv2 modify-listener
            --listener-arn "{{ listener_arn }}"
            --default-actions Type=forward,TargetGroupArn="{{ new_target_group_arn }}"
    ```

6.  **`files/index_blue.html`** and **`files/index_green.html`** with your desired content.

7.  **`Jenkinsfile`** (The final version)
    ```groovy
    // Jenkinsfile
    
    // ---v---v---v---v--- ARNs MASKED FOR SECURITY ---v---v---v---v---
    def BLUE_TG_ARN = "arn:aws:elasticloadbalancing:ap-south-1:************:targetgroup/blue-tg/..."
    def GREEN_TG_ARN = "arn:aws:elasticloadbalancing:ap-south-1:************:targetgroup/green-tg/..."
    def LISTENER_ARN = "arn:aws:elasticloadbalancing:ap-south-1:************:listener/app/my-app-alb/..."
    // ---^---^---^---^--- ARNs MASKED FOR SECURITY ---^---^---^---^---
    
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
                        ansiblePlaybook(
                            playbook: 'traffic_switch.yml',
                            inventory: 'aws_ec2.yml',
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
    ```

---

### **Phase 4: Running Your First Deployment**

**Goal:** Execute the pipeline to deploy the application.

1.  **Create the Jenkins Job:** In Jenkins, create a new "Pipeline" job. Configure it to use "Pipeline script from SCM", select "Git", and provide your repository URL (`https://github.com/ShubhamBhavsar101/AWS-Blue-Green-Deployment`).
2.  **Start the Build:** Click **Build Now**.
3.  **Follow the Pipeline Prompts:**
    *   **Stage 'Live Traffic Check':** The console will print which environment is currently live.
    *   **Stage 'Confirmation to Deploy Code':** The pipeline will **PAUSE**. It will tell you which environment it's about to deploy to. Click **Proceed**.
    *   **Stage 'Configure Inactive Server':** Ansible will run automatically.
    *   **Stage 'Manual Approval to Go Live':** The pipeline will check the health of the new environment. If healthy, it will **PAUSE** again. Click **Proceed** to make the change live.
    *   **Stage 'Switch Live Traffic':** The `traffic_switch.yml` playbook runs automatically, changing the ALB listener rule.
    *   **Stage 'Deployment Complete':** Your new environment is now live.
4.  **Verification:** Open your ALB's DNS name in a browser to see the newly deployed version.
