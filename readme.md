# **Step-by-Step Guide: Containerized CI/CD Pipeline with Jenkins & Docker**
This guide provides a **beginner-friendly**, **step-by-step** walkthrough on how to set up a **Jenkins CI/CD pipeline** that builds, tests, and deploys a containerized application using **Docker**.

---

## **Project Overview**
We will:
 Set up **Jenkins** on an **AWS EC2 instance**  
 Install **Docker** to containerize our application  
 Create a **Jenkins pipeline** to:
   - Clone a **GitHub repository**
   - Build a **Docker image**
   - Run **tests**
   - Push the image to **Docker Hub**
   - Deploy the **latest version of the app**
   - Automate the entire process **using GitHub & Jenkins**
   - Configure and set email notifications **using Mailutils & Jenkins Email Notification**

---

## **Step 1: Set Up AWS EC2 Instance**
### **1️ Launch an EC2 Instance**
- **Instance Type:** `t2.micro` (Free Tier eligible)
- **OS:** `Ubuntu 22.04 LTS`
- **Storage:** At least **10GB**
- **Security Group Rules:**
  - **SSH (22)** → Your IP
  - **HTTP (80)** → Anywhere
  - **Custom TCP (8080)** → Anywhere (for Jenkins)
  - **Custom TCP (5000)** → Anywhere (for our app)

### **2️ Connect to Your EC2 Instance**
```bash
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```
Replace `your-key.pem` with your **private key** and `your-ec2-public-ip` with the **instance's public IP**.

---

## **Step 2: Install Docker**
Jenkins will use Docker to **build and deploy containers**.

### **1️ Install Docker**
```bash
sudo apt update && sudo apt install -y docker.io
```

### **2️ Enable & Start Docker**
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### **3️ Allow Jenkins to Use Docker**
```bash
sudo usermod -aG docker ubuntu
sudo chmod 666 /var/run/docker.sock
```

---

## **Step 3: Install Jenkins**
Jenkins will automate the pipeline.

### **1️ Add Jenkins Repository**
```bash
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
```

### **2️ Install Jenkins**
```bash
sudo apt update && sudo apt install -y openjdk-17-jdk jenkins
```

### **3️ Start Jenkins**
```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

### **4️ Get Jenkins Admin Password**
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
🔹 Copy this password—you’ll need it to **unlock Jenkins**.

---

## **Step 4: Configure Jenkins**
### **1️ Access Jenkins Web UI**
- Open a browser and go to:
  ```
  http://your-ec2-public-ip:8080
  ```
- **Unlock Jenkins** using the password from Step 3️.

### **2️ Install Plugins**
- Select **"Install Suggested Plugins"**.

### **3️ Create an Admin User**
- Set up **username, password, and email**.

### **4️ Install Required Plugins**
- Go to **Manage Jenkins → Manage Plugins → Available Plugins**
- Install:
  - `Pipeline`
  - `Git Plugin`
  - `Docker Pipeline`
  - `Docker Commons`
- Restart Jenkins:
  ```bash
  sudo systemctl restart jenkins
  ```

---

## **Step 5: Create a GitHub Repository**
### **1️ Go to GitHub & Create a New Repo**
- Name it **`jenkins-docker-ci-cd`**
- Make it **Public** (or use credentials for a Private repo).

### **2️ Clone the Repository on the Instance**
```bash
git clone https://github.com/your-github-username/jenkins-docker-ci-cd.git
cd jenkins-docker-ci-cd
```

### **3️ Create Required Files**
```bash
touch Dockerfile app.py requirements.txt Jenkinsfile
mkdir tests
echo "Flask==2.2.5\nWerkzeug==2.2.3\ngunicorn==20.1.0\npytest==7.2.1\nrequests==2.28.2" > requirements.txt
echo "from flask import Flask\napp = Flask(__name__)\n@app.route('/')\ndef home():\n    return 'Hello from Jenkins CI/CD with Docker!'\nif __name__ == '__main__':\n    app.run(host='0.0.0.0', port=5000)" > app.py
echo "def test_sample():\n    assert True" > tests/test_sample.py
```
**Dockerfile**
```bash
# Use an official Python runtime as the base
FROM python:3.9-slim

# Set the working directory
WORKDIR /app

# Copy requirements and install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application files
COPY . .

# Expose the application port
EXPOSE 5000

# Run the application
CMD ["python", "app.py"]
```

**app.py**
```bash
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from Jenkins CI/CD with Docker!, task by neyo55"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)

```

**requirements.txt**
```bash
Flask==2.2.5
Werkzeug==2.2.3
gunicorn==20.1.0
pytest==7.2.1
requests==2.28.2
```

**tests/test_sample.py**
```bash
def test_sample(): assert True
```

**JenKinsfile**
```bash
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "neyo55/jenkins-docker-ci-cd"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                sh 'rm -rf *'
            }
        }

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/neyo55/jenkins-docker-ci-cd.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE:latest .'
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    def testDirExists = sh(script: 'if [ -d "tests" ]; then echo "exists"; else echo "missing"; fi', returnStdout: true).trim()
                    if (testDirExists == "exists") {
                        sh 'docker run --rm $DOCKER_IMAGE pytest tests/'
                    } else {
                        echo "No tests found, skipping test stage."
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withDockerRegistry([credentialsId: 'docker-hub-cred', url: '']) {
                    sh 'docker push $DOCKER_IMAGE:latest'
                }
            }
        }

        stage('Deploy to Test Environment') {
            steps {
                sh '''
                    docker stop test_container || true
                    docker rm test_container || true
                    docker run -d -p 5000:5000 --name test_container neyo55/jenkins-docker-ci-cd:latest
                '''
            }
        }
    }
}
```
---

## **Step 6: Create Credentials and a Jenkins Pipeline Job**
 1 **Go to Jenkins Dashboard → Manage Jenkins**  
 2 **Go to Credentials → global**  
 3 **Create for Github and also Dockerhub** 

![Credentials](./credentials.png)

### **Create a Jenkins Pipeline Job**
 4 **Go to Jenkins Dashboard → New Item**  
 5 **Enter "jenkins-docker-ci-cd"**  
 6 **Select "Pipeline" → Click OK**  
 7 **Scroll to Pipeline Section**

 8 **Definition → "Pipeline script from SCM"**

 9 **SCM → "Git"**

 10 **Repository URL**
   ```
   https://github.com/your-github-username/jenkins-docker-ci-cd.git
   ```
8️ **Branch** → `*/main`
9️ **Click Save & Apply**

---

## **Step 7: Write the Jenkins Pipeline**
Add this to **`Jenkinsfile`**:

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "your-dockerhub-username/jenkins-docker-ci-cd"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                sh 'rm -rf *'
            }
        }

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/your-github-username/jenkins-docker-ci-cd.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE:latest .'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'docker run --rm $DOCKER_IMAGE pytest tests/'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withDockerRegistry([credentialsId: 'docker-hub-cred', url: '']) {
                    sh 'docker push $DOCKER_IMAGE:latest'
                }
            }
        }

        stage('Deploy to Test Environment') {
            steps {
                sh '''
                    docker stop test_container || true
                    docker rm test_container || true
                    docker run -d -p 5000:5000 --name test_container $DOCKER_IMAGE:latest
                '''
            }
        }
    }
}
```

---

## **Step 8: Run the Pipeline**
1️ **Click "Build Now"** in Jenkins  

![Build Page](./build-page.png)
2️ Monitor the **Console Output**  
3️ **Check the Running App**
```bash
curl http://your-ec2-public-ip:5000
```
![The-Page](./Pasted%20image.png)
---


## **Step 9: Configure Email Notifications in Jenkins**
Jenkins can send **email notifications** when a **build succeeds or fails**. This ensures you are **notified** whenever something goes wrong in the pipeline.

This section provides a **detailed step-by-step guide** to **install, configure, and test email notifications** using Gmail SMTP.

---

## **1️⃣ Install and Configure Mail Utilities**
Jenkins requires **mail utilities** to send emails. We will install and configure **Postfix** for this purpose.

### **1. Install Required Mail Utilities**
```bash
sudo apt update && sudo apt install -y mailutils postfix
```
- **Postfix**: The mail transfer agent (MTA) used to send emails.
- **Mailutils**: Provides the `mail` command used for sending emails.

During installation, you will be prompted to choose a **Postfix Configuration Type**. Select:

➡ **"Internet Site"** and press **Enter**  
➡ When prompted for "System mail name," enter your **server's hostname** (or leave it as default).

---

### **2. Configure Postfix for Gmail SMTP**
Now, configure **Postfix** to send emails via **Gmail SMTP**.

1️⃣ Open the Postfix configuration file:
```bash
sudo nano /etc/postfix/main.cf
```

2️⃣ Scroll to the **relayhost** section and update it as follows and please delete any  **relayhost** and  **smtp_tls_security_level** you find in the file before pasting this configuration below to avoid errors. :
```
relayhost = [smtp.gmail.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = encrypt
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
```
3️⃣ Create an authentication file:
```bash
sudo nano /etc/postfix/sasl_passwd
```
4️⃣ Add your Gmail credentials:
```
[smtp.gmail.com]:587 your-sender-gmail@gmail.com:gmail-app-password
```
5️⃣ Save and secure the authentication file:
```bash
sudo chmod 600 /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
```
6️⃣ Restart Postfix:
```bash
sudo systemctl restart postfix
```
---

## **2️⃣ Test Email Sending from the Server**
To verify that **email sending works**, run:
```bash
echo "This is a test email from Jenkins server" | mail -s "Test Email" your-receiver-email@gmail.com
```
- If successful, you should **receive an email** at `your-receiver-email@gmail.com`.
- If the email does **not arrive**, check **spam/junk** folders.
- If there’s an error, check mail logs:
```bash
sudo tail -f /var/log/mail.log
```

Once email sending is confirmed, **proceed to configure Jenkins**.

---

## **3️⃣ Configure Gmail SMTP in Jenkins**
Now, configure **Jenkins** to send emails using **Gmail SMTP**.

1️⃣ **Go to**: **Jenkins Dashboard → Manage Jenkins → System Configure then click System**  
2️⃣ Scroll down to **"Extended E-mail Notification"**  
3️⃣ **Set SMTP Server** and **Set SMTP Port**:  
   ```
   localhost
   ```
   ```
   25
   ```
4️⃣ Click **Advanced** and enter:
   - **Credentials**: `Make sure you have already configured your email credentials under global and then add and select it here`
   - **Use SSL, TLS, OAuth 2.0**: (Unchecked)

5️⃣ **Set Default Email Suffix** (Optional):  
   ```
   @gmail.com
   ```
   **Default Recipients** 
   ```
   your-receiver-email@example.com
   ```
   ### E-mail Notification ###

   **SMTP Server**
   ```
   localhost
   ```

   **Default user e-mail suffix** 
   ```
   @gmail.com
   ```


6️⃣ Check **Test Configuration by sending test e-mail** → Enter your email  
Click **Test Configuration** and if successful.

7️⃣ Click **Save** you should receive alert in your receiver mail.

---

## **4️⃣ Add Email Notifications to the Jenkins Pipeline**
Modify your `Jenkinsfile` by adding a **post-build action** to send email alerts for **build success/failure**:

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "your-dockerhub-username/jenkins-docker-ci-cd"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                sh 'rm -rf *'
            }
        }

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/your-github-username/jenkins-docker-ci-cd.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE:latest .'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'docker run --rm $DOCKER_IMAGE pytest tests/'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withDockerRegistry([credentialsId: 'docker-hub-cred', url: '']) {
                    sh 'docker push $DOCKER_IMAGE:latest'
                }
            }
        }

        stage('Deploy to Test Environment') {
            steps {
                sh '''
                    docker stop test_container || true
                    docker rm test_container || true
                    docker run -d -p 5000:5000 --name test_container $DOCKER_IMAGE:latest
                '''
            }
        }
    }

    post {
        success {
            mail to: 'your-email@gmail.com',
                 subject: "Jenkins Build Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Good job! The Jenkins build for ${env.JOB_NAME} #${env.BUILD_NUMBER} was successful! 🎉"
        }
        failure {
            mail to: 'your-email@gmail.com',
                 subject: "Jenkins Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Oops! The Jenkins build for ${env.JOB_NAME} #${env.BUILD_NUMBER} failed. Please check the logs."
        }
    }
}
```

**Push the new updated jenkins file to your github repository.**

**restart jenkins server**
```
sudo systemctl restart jenkins
```
---

## **5️⃣ Run the Pipeline & Verify Email Notifications**
1️⃣ **Trigger a build** in Jenkins  
2️⃣ If the build **succeeds**, you will receive an email with a **success message**  
3️⃣ If the build **fails**, you will receive an **error alert**  
4️⃣ Check your **inbox/spam folder** for the notification  

✅ **Now, Jenkins will send email notifications automatically after every build!** 

---


So, by configuring email notifications in Jenkins, you will always stay informed about your **CI/CD pipeline's health** without manually checking logs.

---
## **Conclusion**
**Your CI/CD Pipeline is now fully automated!**  
**Every GitHub push triggers Jenkins to build, test, and deploy the app.**

**Receievs Email notifications each time you trigger a build.**

---
