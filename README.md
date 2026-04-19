# Deploy Java Web App on Docker Container via Jenkins on AWS

> **Tự động hóa toàn bộ quy trình Build → Package → Deploy** một ứng dụng Java Web lên Docker Container chạy trên AWS EC2, sử dụng Jenkins CI/CD Pipeline.

---

## 🧰 Tech Stack

| Công nghệ | Phiên bản |
|---|---|
| **AWS EC2** | Amazon Linux 2023 |
| **Java** | Amazon Corretto 21 (LTS) |
| **Maven** | 3.9.15 |
| **Jenkins** | LTS (2.x) |
| **Git** | 2.x |
| **Docker Engine** | 26.x |
| **Tomcat** | 10.1 (trong Docker) |

---

## 📋 Agenda

1. Setup Jenkins Server trên AWS EC2
2. Tích hợp GitHub với Jenkins
3. Tích hợp Maven với Jenkins
4. Setup Docker Host trên EC2
5. Tích hợp Docker với Jenkins
6. Tạo Jenkins Job – Build & Copy Artifact
7. Tự động hóa Build và Deploy toàn diện

---

## ✅ Prerequisites

- Tài khoản AWS (Free Tier đủ dùng)
- Tài khoản GitHub với source code Java Web App
- Máy local có cài SSH client (Terminal, MobaXterm, v.v.)
- Kiến thức cơ bản về Docker và Git

---

## Step 1: Setup Jenkins Server trên AWS EC2

### 1.1 – Khởi tạo EC2 Instance

1. Đăng nhập **AWS Management Console** → **EC2 Dashboard** → **Launch Instance**.
2. Cấu hình:

| Tham số | Giá trị |
|---|---|
| **Name** | `jenkins-server` |
| **AMI** | Amazon Linux 2023 AMI |
| **Instance Type** | `t3.micro` *(Free Tier eligible)* |
| **Key Pair** | Tạo mới hoặc dùng key có sẵn (`.pem`) |
| **Security Group** | Mở port **22** (SSH) và **8080** (Jenkins UI) |

3. Click **Launch Instance** → chờ `Running` → kết nối SSH:

```bash
chmod 400 your-key.pem
ssh -i "your-key.pem" ec2-user@<EC2-PUBLIC-IP>
```

---

### 1.2 – Cài đặt Java 21 (Amazon Corretto)

```bash
sudo dnf update -y
sudo dnf install java-21-amazon-corretto -y

# Kiểm tra
java -version
# Output: openjdk version "21.x.x" ...

# Lấy JAVA_HOME chính xác (lưu lại để dùng ở Step 3)
java -XshowSettings:all -version 2>&1 | grep "java.home"
# Output ví dụ: java.home = /usr/lib/jvm/java-21-amazon-corretto.x86_64
```

---

### 1.3 – Cài đặt Jenkins

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo

sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

sudo dnf install jenkins -y

sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

---

### 1.4 – Truy cập Jenkins Web UI

Mở trình duyệt: `http://<EC2-PUBLIC-IP>:8080`

```bash
# Lấy mật khẩu Admin lần đầu
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Dán mật khẩu → **Install suggested plugins** → tạo tài khoản Admin.

---

### 1.5 – Cấu hình Executor (Quan trọng!)

Nếu build bị kẹt **"Waiting for next available executor"**:

1. **Manage Jenkins** → **Nodes** → **Built-In Node** → **Configure**
2. Đặt **Number of executors** = `2` → **Save**
3. Vào lại **Nodes** → **Built-In Node** → click **"Bring this node back online"**

**Nếu node offline do /tmp đầy** (SSH vào Jenkins Server EC2):

```bash
sudo rm -rf /tmp/*
sudo dnf clean all
df -h /tmp
```

Sau đó: **Manage Jenkins** → **Nodes** → **Built-In Node** → **Configure** → tìm **Free Temp Space Threshold** → đổi xuống `900MB` → **Save** → click **"Bring this node back online"**.

---

## Step 2: Tích hợp GitHub với Jenkins

### 2.1 – Cài đặt Git trên Jenkins EC2

```bash
sudo dnf install git -y
git --version
```

### 2.2 – Cài Plugin GitHub Integration

**Manage Jenkins** → **Plugins** → **Available plugins** → tìm `GitHub Integration` → **Install**.

### 2.3 – Cấu hình Git trong Jenkins

**Manage Jenkins** → **Tools** → **Git installations**:
- **Name:** `git`
- **Path to Git executable:** `git`

→ **Save**

---

## Step 3: Tích hợp Maven với Jenkins

### 3.1 – Cài đặt Maven 3.9.15

> ⚠️ Phải dùng domain `dlcdn.apache.org`, KHÔNG dùng `downloads.apache.org` (chỉ lưu bản mới nhất, các bản cũ trả về 404). Luôn kiểm tra bản mới nhất tại https://maven.apache.org/download.cgi

```bash
cd /opt

sudo wget https://dlcdn.apache.org/maven/maven-3/3.9.15/binaries/apache-maven-3.9.15-bin.tar.gz

sudo tar -xvzf apache-maven-3.9.15-bin.tar.gz

sudo mv apache-maven-3.9.15 maven

sudo rm apache-maven-3.9.15-bin.tar.gz
```

### 3.2 – Thiết lập Environment Variables

```bash
sudo tee /etc/profile.d/maven.sh > /dev/null <<'EOF'
export M2_HOME=/opt/maven
export M2=$M2_HOME/bin
export JAVA_HOME=/usr/lib/jvm/java-21-amazon-corretto.x86_64
export PATH=$M2:$JAVA_HOME/bin:$PATH
EOF

source /etc/profile.d/maven.sh

# Kiểm tra
mvn -version
# Output: Apache Maven 3.9.15 ...
```

> 💡 Thay `java-21-amazon-corretto.x86_64` bằng path chính xác lấy từ Step 1.2.

### 3.3 – Cấu hình Maven trong Jenkins

1. **Manage Jenkins** → **Plugins** → tìm `Maven Integration` → **Install**
2. **Manage Jenkins** → **Tools**:
   - **JDK installations** → **Add JDK**:
     - **Name:** `java-21`
     - Bỏ tick **Install automatically**
     - **JAVA_HOME:** điền path lấy từ Step 1.2 *(ví dụ: `/usr/lib/jvm/java-21-amazon-corretto.x86_64`)*
     > ℹ️ Nếu Jenkins hiện **cảnh báo vàng** *"is not a directory on the Jenkins controller (but perhaps it exists on some agents)"* → **bình thường, bấm Save tiếp, không phải lỗi**.
   - **Maven installations** → **Add Maven**:
     - **Name:** `maven-3.9.15`
     - Tick **Install automatically** → chọn version `3.9.15`
3. **Save**

---

## Step 4: Setup Docker Host trên EC2

### 4.1 – Tạo EC2 Instance mới

Tạo instance tên `docker-host` (cấu hình tương tự Step 1.1). Trong Security Group, mở thêm port range **8081–9000**.

### 4.2 – Cài đặt Docker Engine

```bash
sudo dnf update -y
sudo dnf install docker -y
sudo systemctl enable docker
sudo systemctl start docker
docker --version
```

### 4.3 – Tạo user `dockeradmin`

```bash
sudo useradd dockeradmin
sudo passwd dockeradmin       # nhập và ghi nhớ mật khẩu

sudo usermod -aG docker dockeradmin
```

### 4.4 – Thêm ec2-user vào nhóm docker

```bash
sudo usermod -aG docker ec2-user
newgrp docker
```

### 4.5 – Bật xác thực bằng mật khẩu SSH

```bash
sudo sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl reload sshd
```

### 4.6 – Tạo thư mục và Dockerfile

```bash
sudo mkdir -p /opt/docker
sudo chown -R dockeradmin:dockeradmin /opt/docker
```

Tạo Dockerfile (phải dùng `sudo` vì `ec2-user` không sở hữu thư mục):

```bash
sudo nano /opt/docker/Dockerfile
```

Nội dung Dockerfile:

```dockerfile
FROM tomcat:10.1

# Fix issue: webapps trống trong image Tomcat 10+
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps

# Copy WAR artifact từ Jenkins build
COPY ./*.war /usr/local/tomcat/webapps

EXPOSE 8080
```

Lưu file: **Ctrl+O** → **Enter** → **Ctrl+X**

> ⚠️ Nếu ứng dụng dùng `javax.*` (Java EE cũ), đổi sang `tomcat:9.0`.

### 4.7 – Build và chạy thử container

```bash
docker build -t tomcatserver:latest /opt/docker/
docker run -d --name tomcat-test -p 8085:8080 tomcatserver:latest
docker ps
```

Lấy Public IP hiện tại của Docker Host:

```bash
curl ifconfig.me
```

Truy cập: `http://<DOCKER-HOST-IP>:8085`

> 💡 EC2 thay đổi Public IP mỗi lần stop/start. Luôn chạy `curl ifconfig.me` để lấy IP mới nhất.

---

## Step 5: Tích hợp Docker với Jenkins

### 5.1 – Cài Plugin "Publish Over SSH"

**Manage Jenkins** → **Plugins** → **Available plugins** → tìm `Publish Over SSH` → **Install**.

### 5.2 – Cấu hình SSH Server

**Manage Jenkins** → **System** → cuộn đến **Publish over SSH** → **Add SSH Server**:

| Trường | Giá trị |
|---|---|
| **Name** | `docker-host` |
| **Hostname** | Private IP của Docker Host EC2 |
| **Username** | `dockeradmin` |
| **Remote Directory** | `/opt/docker` |

Click **Advanced** → tick **Use password authentication** → nhập mật khẩu `dockeradmin`.

Click **Test Configuration** → phải thấy `Success` → **Apply** → **Save**.

---

## Step 6: Tạo Jenkins Job – Build & Copy Artifact

1. **New Item** → tên `BuildAndDeploy` → chọn **Maven project** → **OK**
2. Cấu hình:
   - **Source Code Management:** Git → nhập URL repo GitHub
   - **Build Triggers:** tick `Poll SCM` → schedule `* * * * *`
   - **Build Section:**
     - **Root POM:** `hello-world/pom.xml`
     - **Goals:** `clean install`
3. **Post-build Actions** → **Send build artifacts over SSH**:

| Trường | Giá trị |
|---|---|
| **SSH Server Name** | `docker-host` |
| **Source files** | `hello-world/webapp/target/webapp.war` |
| **Remove prefix** | `hello-world/webapp/target` |
| **Remote directory** | `//opt//docker` |

4. **Apply** → **Save** → **Build Now**

Kiểm tra trên Docker Host sau khi build xong:

```bash
ls /opt/docker/
# Phải thấy: Dockerfile  webapp.war
```

---

## Step 7: Tự động hóa Build và Deploy toàn diện

### 7.1 – Cập nhật Exec Command

Vào **BuildAndDeploy** → **Configure** → **Send build artifacts over SSH** → **Exec command**:

```bash
cd /opt/docker

# Dừng và xoá container cũ nếu tồn tại
docker stop regapp 2>/dev/null || true
docker rm regapp 2>/dev/null || true

# Build image mới
docker build -t regapp:latest .

# Chạy container mới
docker run -d --name regapp -p 8087:8080 regapp:latest
```

**Apply** → **Save**

### 7.2 – Kích hoạt Pipeline

Commit bất kỳ thay đổi nào lên GitHub. Jenkins tự động:

1. ✅ Phát hiện thay đổi qua Poll SCM
2. ✅ Clone code từ GitHub
3. ✅ Build `.war` bằng Maven
4. ✅ Copy `webapp.war` sang Docker Host qua SSH
5. ✅ Build Docker image mới
6. ✅ Chạy container mới trên port 8087

**Nếu gặp lỗi `Exec exit status not zero. Status [126]`** — `dockeradmin` thiếu quyền Docker:

```bash
# SSH vào Docker Host
sudo usermod -aG docker dockeradmin
sudo systemctl restart docker
```

Sau đó **Build Now** lại trên Jenkins.

### 7.3 – Kiểm tra kết quả

```bash
# Trên Docker Host
docker ps
# Phải thấy container "regapp" đang chạy port 8087

docker logs regapp
```

Truy cập ứng dụng:

```
http://<DOCKER-HOST-IP>:8087/webapp/
```

---

## 🐛 Bảng xử lý lỗi thường gặp

| Lỗi | Nguyên nhân | Cách fix |
|---|---|---|
| `ERROR 404` khi tải Maven | `downloads.apache.org` không lưu bản cũ | Dùng `dlcdn.apache.org` + kiểm tra version mới nhất |
| `Source option 7 is no longer supported` | `pom.xml` set source/target = `1.7` | Đổi thành `<source>1.8</source><target>1.8</target>` |
| `Waiting for next available executor` | Executor = 0 hoặc node offline | Đặt executor = 2, bấm "Bring node back online" |
| Node offline do `/tmp` đầy | `/tmp` < 1GB threshold mặc định | Dọn `/tmp`, hạ threshold xuống `900MB` |
| `doesn't look like a JDK directory` (lỗi đỏ) | JAVA_HOME sai | Chạy `java -XshowSettings:all -version 2>&1 \| grep java.home` |
| `is not a directory on Jenkins controller` (cảnh báo vàng) | Jenkins controller không tìm thấy path | **Bình thường** – bấm Save tiếp |
| `Status [126]` SSH exec failed | `dockeradmin` không có quyền docker | `sudo usermod -aG docker dockeradmin && sudo systemctl restart docker` |
| Web `can't be reached` | IP đổi sau khi restart EC2 | `curl ifconfig.me` lấy IP mới |
| `/opt/docker/` không có `webapp.war` | Source files path sai trong Jenkins | Source=`hello-world/webapp/target/webapp.war`, Remove prefix=`hello-world/webapp/target` |
| `permission denied` khi nano Dockerfile | `ec2-user` không sở hữu `/opt/docker` | Dùng `sudo nano /opt/docker/Dockerfile` |
| `HTTP Status 404` trên webapp | WAR chưa vào container | Kiểm tra `docker exec regapp ls /usr/local/tomcat/webapps/` |

---

## 🧹 Dọn dẹp tài nguyên

```bash
# Trên Docker Host
docker stop $(docker ps -q)
docker image prune -a
```

Vào **EC2 Console** → chọn tất cả instance → **Instance State** → **Terminate**.

---

## 🔐 Best Practices (Production)

- Dùng **SSH Key** thay mật khẩu cho `dockeradmin`
- Lưu secrets trong **Jenkins Credentials Store**
- Dùng **Jenkinsfile** (Pipeline as Code) để version control pipeline
- Tag Docker image theo build number: `regapp:${BUILD_NUMBER}`
- Cân nhắc dùng **Amazon ECR** thay vì build image trực tiếp trên host
- Dùng **Elastic IP** để tránh Public IP thay đổi mỗi lần restart EC2

---

## 📌 Tóm tắt Kiến trúc

```
Developer
    │
    │ git push
    ▼
GitHub Repository
    │
    │ Poll SCM (mỗi phút)
    ▼
Jenkins Server (EC2 - Amazon Linux 2023)
  ├── Java 21 (Amazon Corretto)
  ├── Maven 3.9.15
  └── Build → webapp.war
    │
    │ Publish Over SSH (dockeradmin@docker-host)
    ▼
Docker Host (EC2 - Amazon Linux 2023)
  ├── Docker Engine 26.x
  ├── /opt/docker/Dockerfile + webapp.war
  ├── docker build -t regapp:latest
  └── docker run -p 8087:8080 regapp:latest
    │
    ▼
http://<docker-host-ip>:8087/webapp/
```
