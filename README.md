# AWS-Blue-Green-Deployment

This guide will be broken into three phases:
*   **Phase 1: One-Time AWS Setup.** We will build our entire foundation. You only do this once.
*   **Phase 2: Prepare Automation Code.** We will create the simple files for Ansible and Jenkins.
*   **Phase 3: Run the First Deployment.** We will use our pipeline to deploy the application.

---

### **Phase 1: The One-Time Setup - Building Your Foundation**

**Goal:** Create everything you need in AWS. By the end of this phase, you will have two servers running, registered to a load balancer, and ready for deployment.

#### **Step 1: Create Security Groups (Your Virtual Firewalls)**

1.  Navigate to the **EC2** service in the AWS Console.
2.  On the left menu, go to `Network & Security` -> `Security Groups`.
3.  **Create the Load Balancer's Firewall:**
    *   Click `Create security group`.
    *   **Name:** `alb-sg`
    *   **Description:** `Allows internet web traffic`
    *   **Inbound rules:** Add a rule for `HTTP` with the Source as `Anywhere-IPv4` (`0.0.0.0/0`).
    *   Click `Create security group`.
4.  **Create the Servers' Firewall:**
    *   Click `Create security group` again.
    *   **Name:** `app-sg`
    *   **Description:** `Allows traffic only from the ALB`
    *   **Inbound rules:** Add a rule for `HTTP`. For the Source, select the `alb-sg` group you just created.
    *   Click `Create security group`.

**Checkpoint:** You now have two security groups: `alb-sg` and `app-sg`.

#### **Step 2: Create the EC2 Instances (Your Blue & Green Servers)**

1.  In the EC2 service, go to **Instances** on the left menu, then click **Launch instances**.
2.  **Launch the BLUE Server:**
    *   **Name:** `blue-server`
    *   **Application and OS Images:** Select **Amazon Linux** (e.g., Amazon Linux 2023).
    *   **Instance type:** `t2.micro`.
    *   **Key pair:** Select your key pair.
    *   **Network settings:** Click `Edit`. Under `Security group`, choose `Select existing security group` and pick your **`app-sg`**.
    *   **Tags:** Expand the `Advanced details`. Scroll to the bottom and under `Tags`, add a tag with **Key:** `Deployment` and **Value:** `Blue`.
    *   Click **Launch instance**.
3.  **Launch the GREEN Server:**
    *   Click **Launch instances** again.
    *   Follow the exact same steps, but with these two changes:
        *   **Name:** `green-server`
        *   **Tags:** Add a tag with **Key:** `Deployment` and **Value:** `Green`.
    *   Click **Launch instance**.

**Checkpoint:** You now have two running EC2 instances: `blue-server` and `green-server`.

#### **Step 3: Create Target Groups & Register Your Servers**

1.  In the EC2 service menu, go to `Load Balancing` -> **Target Groups**.
2.  **Create and Configure the BLUE Group:**
    *   Click `Create target group`.
    *   **Choose a target type:** `Instances`.
    *   **Target group name:** `blue-tg`.
    *   Click `Next`.
    *   On the "Register targets" page, check the box next to your **`blue-server`** and click `Include as pending below`.
    *   Click `Create target group`.
3.  **Create and Configure the GREEN Group:**
    *   Click `Create target group` again.
    *   Follow the same steps, but name it `green-tg` and register your **`green-server`** to it.

**Checkpoint:** You have two target groups. If you click on `blue-tg` -> `Targets`, you should see `blue-server`. The same applies to `green-tg` and `green-server`. The status might show as `unhealthy` for now, which is normal.

#### **Step 4: Create the Application Load Balancer (The Traffic Cop)**

1.  In the EC2 service menu, go to `Load Balancing` -> **Load Balancers**.
2.  Click **Create Load Balancer**, then choose **Application Load Balancer**.
3.  **Configure it:**
    *   **Load balancer name:** `my-app-alb`
    *   **Network mapping:** Select your VPC and check at least two subnets.
    *   **Security groups:** Deselect `default` and select your **`alb-sg`**.
    *   **Listeners and routing:** The listener should be `HTTP` on port `80`. For the `Default action`, select your **`blue-tg`**.
4.  Click **Create load balancer**.

**Foundation Complete!** You have built your entire environment. The load balancer is now sending traffic to your `blue-server`, which is waiting to be configured.

---

### **Phase 2: Prepare Your Automation Code**

**Goal:** Create the simple files that Ansible and Jenkins will use.

1.  **Set Up Your Project Folder:** On your computer, create this structure:
    ```
    my-blue-green-project/
    ├── Jenkinsfile
    ├── configure_blue.yml
    ├── configure_green.yml
    └── files/
        ├── index_blue.html
        └── index_green.html
    ```
2.  **Create HTML Files:**
    *   In `files/index_blue.html`, add: `<h1>Welcome - I am the BLUE Server!</h1>`
    *   In `files/index_green.html`, add: `<h1>Welcome - I am the GREEN Server!</h1>`
3.  **Create Ansible Playbooks:**
    *   `configure_blue.yml` (This finds the 'Blue' server and installs the web server):
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
    *   `configure_green.yml` (This finds the 'Green' server and installs the web server):
        ```yaml
        - name: Configure the GREEN Server
          hosts: tag_Deployment_Green
          become: yes
          tasks:
            - name: Install httpd
              dnf: { name: httpd, state: present }
            - name: Copy index file
              copy: { src: files/index_green.html, dest: /var/www/html/index.html }
            - name: Start httpd
              service: { name: httpd, state: started, enabled: yes }
        ```
4.  **Create the Jenkinsfile:**
    *   **IMPORTANT:** Go to your AWS console and get the ARNs (Amazon Resource Names) for your two target groups and your load balancer's listener. You will need to paste them into the first three lines of this file.
    *   Copy this code into your `Jenkinsfile`:
        ```groovy
        // ---> PASTE YOUR REAL ARNs HERE <---
        def BLUE_TG_ARN = "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/blue-tg/..."
        def GREEN_TG_ARN = "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/green-tg/..."
        def LISTENER_ARN = "arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/my-app-alb/..."

        pipeline {
            agent any
            environment {
                AWS_ACCESS_KEY_ID     = credentials('aws-access-key')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
                AWS_DEFAULT_REGION    = 'us-east-1' // Change if your region is different
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
        ```

---

### **Phase 3: Run Your First Deployment**

**Goal:** Use your new pipeline to deploy your first application version.

1.  **Set up the Pipeline in Jenkins:** Create a new "Pipeline" job in Jenkins and point it to this `Jenkinsfile` in your source code repository.
2.  **Start the Job:** Click **Build Now**.
3.  **Stages 1 & 2 (Automatic):** Jenkins will automatically determine that "Green" is the inactive environment and run the `configure_green.yml` playbook on your `green-server`.
4.  **Stage 3 - Your Turn! (Manual Verification):**
    *   The pipeline will **PAUSE**.
    *   **Your Action:** Go to the AWS Console -> EC2 -> Target Groups -> `green-tg`. Click the `Targets` tab and refresh until the `green-server` status is **`healthy`**.
    *   Go back to Jenkins and click **Proceed**.
5.  **Stage 4 - Your Turn! (Switch Traffic):**
    *   The pipeline will **PAUSE** again.
    *   **Your Action:** Go to the AWS Console -> EC2 -> Load Balancers -> select `my-app-alb`. Click the `Listeners` tab. Select the rule and click `Edit`. Change the `Forward to` action to **`green-tg`** and save.
    *   **Test It!** Copy your ALB's DNS name into a new browser tab. You should see "**Welcome - I am the GREEN Server!**"
    *   Go back to Jenkins and click **Proceed**.
6.  **Stage 5 (Automatic):** The pipeline finishes successfully.

**Congratulations!** You have completed a full deployment. If you run the Jenkins job again, it will guide you through configuring the `blue-server` and switching traffic back to it.
