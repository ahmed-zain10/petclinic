
#  Jenkins CI/CD with GitHub Webhook (Spring Petclinic Example)

## 📌 Prerequisites
- **Jenkins** (منزل ومشتغل على بورت 8088)  
- **Docker** + **Docker Compose**  
- **Maven** + **JDK 17** (متضبطين من Jenkins global tools)  
- GitHub repo: [ahmed-zain10/petclinic](https://github.com/ahmed-zain10/petclinic)  
- Cloudflare Tunnel (عشان تفتح Jenkins للعالم الخارجي) → [Cloudflare Tunnel Docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)  

---

##  Jenkins Setup

### 1. Run Jenkins on Port 8088
بعد ما تنزل Jenkins:  
```bash
http://localhost:8088
```

```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
```
وتفتح من المتصفح:  
[http://localhost:8088](http://localhost:8088)

---

### 2. Install Required Plugins
**Manage Jenkins → Plugins**  
ثبت:  
- GitHub plugin  
- Pipeline plugin  

---

### 3. Configure Jenkins URL
**Manage Jenkins → Configure System → Jenkins Location**  
حط الـ URL العام:  
```
https://ownership-bite-graph-full.trycloudflare.com/
```
(اللي طالع من Cloudflare Tunnel)  

---

### 4. Add GitHub Credentials
**Manage Jenkins → Credentials → System → Global credentials (unrestricted) → Add Credentials**  

- Kind: **Secret text**  
- Secret: **Personal Access Token (PAT)** من GitHub  
- ID: `github-token`  
- Description: `GitHub Access Token`  

---

### 5. Configure GitHub Server
**Manage Jenkins → Configure System → GitHub servers → Add GitHub Server**  
- API endpoint: `https://api.github.com`  
- Name: `GitHub.com`  
- Credentials: اختر `github-token` اللي ضفته  
- اضغط Test connection (لازم يديك ✅).  

---

### 6. Create Pipeline Job
- New Item → Pipeline  
- Name: `spring-petclinic-pipeline`  
- Pipeline script → الصق الكود:

```groovy
pipeline {
    agent any

    tools {
        maven 'Maven3'
        jdk 'Java17'
    }

    environment {
        COMPOSE_FILE = 'docker-compose.yml'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/ahmed-zain10/petclinic.git',
                    credentialsId: 'github-token'
            }
        }

        stage('Build JAR') {
            steps {
                dir('spring-petclinic') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Start with Docker Compose') {
            steps {
                dir('spring-petclinic') {
                    sh 'docker compose up -d'
                }
            }
        }

        stage('Verify') {
            steps {
                sh '''
                for i in $(seq 1 30); do
                  if curl -f http://localhost:7070; then
                    echo "App is up!"
                    exit 0
                  fi
                  echo "App not ready yet... retrying in 150s"
                  sleep 150
                done
                echo "App did not start!"
                exit 1
                '''
            }
        }
    }
}
```
![Screenshot 1](Screenshots/1.PNG)
---

## 🔗 GitHub Webhook Setup

1. افتح: [ahmed-zain10/petclinic](https://github.com/ahmed-zain10/petclinic)  
2. Settings → Webhooks → Add webhook  
3. حط:  
   - **Payload URL**:  
     ```
     https://ownership-bite-graph-full.trycloudflare.com/github-webhook/
     ```
   - **Content type**: `application/json`  
   - **Trigger**: Just the push event  
4. Save → جرّب Push → لازم يبان Build جديد في Jenkins  

---
![Screenshot 3](Screenshots/3.PNG)

## 🐳 docker-compose.yml

```yaml
services:
  mysql:
    image: mysql:9.2
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=
      - MYSQL_ALLOW_EMPTY_PASSWORD=true
      - MYSQL_USER=petclinic
      - MYSQL_PASSWORD=petclinic
      - MYSQL_DATABASE=petclinic
    volumes:
      - "./conf.d:/etc/mysql/conf.d:ro"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  petclinic:
    build: .
    depends_on:
      mysql:
        condition: service_healthy
    ports:
      - "7070:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/petclinic
      - SPRING_DATASOURCE_USERNAME=petclinic
      - SPRING_DATASOURCE_PASSWORD=petclinic
```

---

## ✅ Testing

1. اعمل commit + push:  
   ```bash
   git add .
   git commit -m "trigger pipeline"
   git push origin main
   ```
2. GitHub Webhook → Jenkins يشتغل أوتوماتيك  
3. افتح [http://localhost:7070](http://localhost:7070) وشوف التطبيق  


![Screenshot 4](Screenshots/4.PNG)
![Screenshot 2](Screenshots/2.PNG)

---
## 🛠️ Troubleshooting
- Webhook مبيوصلش → تأكد Payload URL ينتهي بـ `/github-webhook/`  
- خطأ 403 → عدّل Jenkins URL في **Configure System**  
- Verify فاشلة → راجع لوجات الكونتينر:  
  ```bash
  docker logs petclinic-app
  ```
