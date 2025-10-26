# ✅ GitHub → Jenkins → SonarQube → Nexus → S3 (optional) → Ansible → Tomcat1 & Tomcat2  

Project: Java Maven Web App (.war)
Target servers: 2 Tomcat servers (load-balanced manually)
All automation: Jenkins pipeline

🧱 STEP 1: AWS Setup

Launch 5 EC2 Instances:

Instance Name	OS	Type	Purpose	Ports to open
ansible-jenkins	Amazon Linux 2023	t2.micro	Jenkins + Ansible controller	22, 8080
sonar	Amazon Linux 2023	t2.medium	SonarQube	22, 9000
nexus	Ubuntu 24.04	t2.medium	Nexus Repository	22, 8081
tomcat1	Amazon Linux 2023	t2.micro	Application Server 1	22, 8080
tomcat2	Amazon Linux 2023	t2.micro	Application Server 2	22, 8080

🔹 Security Groups: Allow inbound from your IP for all necessary ports.

⚙️ STEP 2: Ansible + Jenkins Server Setup
SSH करा:
ssh -i mykey.pem ec2-user@<ANSIBLE_JENKINS_PUBLIC_IP>
sudo -i

System Update
yum update -y

Install Java, Git, Maven
amazon-linux-extras install java-openjdk11 -y
yum install -y git maven wget unzip python3 python3-pip

Install Ansible
yum install -y ansible
ansible --version

Install Jenkins
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
yum install -y jenkins
systemctl enable --now jenkins


✅ Jenkins URL: http://<ANSIBLE_IP>:8080
Get initial password:

cat /var/lib/jenkins/secrets/initialAdminPassword

🐳 STEP 3: SonarQube Setup
SSH into Sonar:
ssh -i mykey.pem ec2-user@<SONAR_IP>
sudo -i

Install Java + Sonar
amazon-linux-extras install java-openjdk11 -y
cd /opt
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.9.6.50800.zip
unzip sonarqube-8.9.6.50800.zip
mv sonarqube-8.9.6.50800 sonarqube
useradd sonar
chown -R sonar:sonar /opt/sonarqube
su - sonar
/opt/sonarqube/bin/linux-x86-64/sonar.sh start


✅ URL: http://<SONAR_IP>:9000
Login: admin / admin

📦 STEP 4: Nexus Repository Setup
SSH into Nexus:
ssh -i mykey.pem ubuntu@<NEXUS_IP>
sudo -i

Install Java + Nexus
apt update -y
apt install -y openjdk-17-jdk wget unzip
cd /opt
wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz
tar -zxvf latest-unix.tar.gz
mv nexus-* nexus
adduser --system --no-create-home --group --disabled-login nexus
chown -R nexus:nexus /opt/nexus /opt/sonatype-work

Setup Service
echo 'run_as_user="nexus"' > /opt/nexus/bin/nexus.rc
cat <<EOF | tee /etc/systemd/system/nexus.service
[Unit]
Description=nexus service
After=network.target
[Service]
Type=forking
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
User=nexus
Restart=on-abort
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now nexus


✅ URL: http://<NEXUS_IP>:8081
Admin password:

cat /opt/sonatype-work/nexus3/admin.password


Create Maven Hosted Repository → Name: hotstarapp

🧩 STEP 5: Tomcat Server Setup (via Ansible)

On Ansible-Jenkins server, create inventory:

vi /etc/ansible/hosts


Add:

[prod]
<tomcat1-private-ip>
<tomcat2-private-ip>


Now create Ansible playbook /etc/ansible/tomcat.yml:

---
- name: Install Tomcat
  hosts: prod
  become: yes
  tasks:
    - name: Install required packages
      yum:
        name:
          - wget
          - java-17-amazon-corretto
        state: present

    - name: Download Tomcat
      get_url:
        url: "https://dlcdn.apache.org/tomcat/tomcat-11/v11.0.10/bin/apache-tomcat-11.0.10.tar.gz"
        dest: /opt/apache-tomcat-11.0.10.tar.gz

    - name: Extract Tomcat
      unarchive:
        src: /opt/apache-tomcat-11.0.10.tar.gz
        dest: /opt/
        remote_src: yes

    - name: Rename Folder
      command: mv /opt/apache-tomcat-11.0.10 /opt/tomcat
      args:
        removes: /opt/apache-tomcat-11.0.10

    - name: Create Tomcat user config
      copy:
        dest: /opt/tomcat/conf/tomcat-users.xml
        content: |
          <tomcat-users>
            <role rolename="manager-gui"/>
            <role rolename="manager-script"/>
            <user username="admin" password="admin123" roles="manager-gui,manager-script"/>
          </tomcat-users>

    - name: Start Tomcat
      command: /opt/tomcat/bin/startup.sh


Run it:

ansible-playbook /etc/ansible/tomcat.yml -i /etc/ansible/hosts


✅ Check:
http://<TOMCAT1_IP>:8080
http://<TOMCAT2_IP>:8080

🧠 STEP 6: Jenkins Configuration

Go to Manage Jenkins → Plugins → Install:

Maven Integration

SonarQube Scanner

Nexus Artifact Uploader

Ansible

Pipeline

Global Tools

Maven → name: Maven

JDK → Java 11

Ansible → /usr/bin/ansible-playbook

Credentials
ID	Type	Description
nexuscreds	Username/Password	Nexus admin credentials
sonartoken	Secret text	Sonar token
linuxcreds	SSH key	For Ansible to deploy
git-creds	Username/Password or none	GitHub access if private repo
🧾 STEP 7: Jenkinsfile (Pipeline Script)

Put this in your GitHub repo root as Jenkinsfile:

pipeline {
    agent any
    environment {
        MVN_HOME = tool name: 'Maven', type: 'hudson.tasks.Maven$MavenInstallation'
        NEXUS_URL = 'http://<NEXUS_IP>:8081'
        NEXUS_REPO = 'hotstarapp'
        SONARQUBE = 'SonarQube'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git url: 'https://github.com/<your-username>/<your-repo>.git', branch: 'main'
            }
        }

        stage('Build with Maven') {
            steps {
                sh "${MVN_HOME}/bin/mvn clean package -DskipTests"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE}") {
                    sh "${MVN_HOME}/bin/mvn sonar:sonar -Dsonar.login=${SONAR_TOKEN}"
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                nexusArtifactUploader artifacts: [[artifactId: 'hotstar', classifier: '', file: 'target/*.war', type: 'war']],
                    credentialsId: 'nexuscreds',
                    groupId: 'in.hotstar',
                    nexusUrl: "${NEXUS_URL}",
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    repository: "${NEXUS_REPO}",
                    version: '1.0'
            }
        }

        stage('Deploy using Ansible') {
            steps {
                ansiblePlaybook credentialsId: 'linuxcreds',
                    disableHostKeyChecking: true,
                    installation: 'ansible',
                    inventory: '/etc/ansible/hosts',
                    playbook: '/etc/ansible/deploy.yml'
            }
        }
    }
}


Create /etc/ansible/deploy.yml:

---
- name: Deploy WAR file to Tomcat
  hosts: prod
  become: yes
  tasks:
    - name: Copy WAR to Tomcat webapps
      copy:
        src: /var/lib/jenkins/workspace/<JOBNAME>/target/*.war
        dest: /opt/tomcat/webapps/
        mode: '0644'

🧩 STEP 8: Run the Pipeline

Open Jenkins → New Item → Pipeline → name it HotstarAppPipeline.

Choose “Pipeline script from SCM”.

SCM: Git → your repo URL → branch: main.

Save & Build Now.

✅ Watch stages:

Checkout ✅
Build ✅
SonarQube ✅
Nexus Upload ✅
Ansible Deploy ✅

🌐 STEP 9: Verify Deployment

Go to:

http://<TOMCAT1_IP>:8080/<your-war-name>/
http://<TOMCAT2_IP>:8080/<your-war-name>/


If both show your web app → 🎉 SUCCESS!
You just built a full CI/CD pipeline from GitHub to AWS via Jenkins, SonarQube, Nexus, and Ansible.

🧭 BONUS: PIPELINE DIAGRAM (TEXT VIEW)
[GitHub] 
   ↓
[Jenkins Pipeline]
   ├── Build (Maven)
   ├── Test (JUnit)
   ├── SonarQube Code Analysis
   ├── Upload WAR → Nexus
   ├── (Optional) Upload → S3
   └── Deploy → Tomcat1 & Tomcat2 via Ansible
