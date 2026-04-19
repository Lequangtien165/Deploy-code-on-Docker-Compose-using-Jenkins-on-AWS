# 🚀 Deploy Java Web App on Docker Container via Jenkins on AWS

> **Tự động hóa toàn bộ quy trình Build → Package → Deploy** một ứng dụng Java Web lên Docker Container chạy trên AWS EC2, sử dụng Jenkins CI/CD Pipeline.

---

## 🧰 Tech Stack

| Công nghệ | Phiên bản | Ghi chú |
|---|---|---|
| **AWS EC2** | Amazon Linux 2023 | Thay thế Amazon Linux 2 |
| **Java** | Amazon Corretto 21 (LTS) | Thay thế Java 11 |
| **Maven** | 3.9.9 | Build tool cho Java |
| **Jenkins** | LTS (2.x) | CI/CD Automation |
| **Git** | 2.x | Source Control |
| **Docker Engine** | 26.x | Container Runtime |
| **Tomcat** | 10.1 (trong Docker) | Java Web Server |

---

## 📋 Agenda

1. Setup Jenkins Server trên AWS EC2
2. Tích hợp GitHub với Jenkins
3. Tích hợp Maven với Jenkins
4. Setup Docker Host trên EC2
5. Tích hợp Docker với Jenkins
6. Tự động hóa Build & Deploy qua Jenkins Pipeline
7. Kiểm tra kết quả

---

## ✅ Prerequisites

- Tài khoản AWS (Free Tier đủ dùng)
- Tài khoản GitHub với source code Java Web App
- Máy local có cài SSH client (Terminal, MobaXterm, v.v.)
- Kiến thức cơ bản về Docker và Git

---

## Step 1: Setup Jenkins Server trên AWS EC2

### 1.1 – Khởi tạo EC2 Instance

1. Đăng nhập **AWS Management Console** → mở **EC2 Dashboard** → click **Launch Instance**.
2. Cấu hình như sau:

| Tham số | Giá trị khuyến nghị |
|---|---|
| **Name** | `jenkins-server` |
| **AMI** | Amazon Linux 2023 AMI |
| **Instance Type** | `t3.micro` *(Free Tier eligible)* |
| **Key Pair** | Tạo mới hoặc dùng cặp key có sẵn (`.pem`) |
| **Security Group** | Mở port **22** (SSH) và **8080** (Jenkins UI) |

> ⚠️ **Lưu ý bảo mật:** Chỉ cho phép IP của bạn truy cập port 22. Không mở `0.0.0.0/0` cho SSH trong môi trường production.

3. Click **Launch Instance** và chờ trạng thái chuyển sang `Running`.
4. Kết nối SSH:

```bash
chmod 400 your-key.pem
ssh -i "your-key.pem" ec2-user@<EC2-PUBLIC-IP>
```

---

### 1.2 – Cài đặt Java 21 (Amazon Corretto)

```bash
# Cập nhật hệ thống
sudo dnf update -y

# Cài đặt Amazon Corretto 21
sudo dnf install java-21-amazon-corretto -y

# Kiểm tra phiên bản
java -version
# Output: openjdk version "21.x.x" ...
```

---

### 1.3 – Cài đặt Jenkins

```bash
# Thêm Jenkins repository
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo

# Import GPG key
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

# Cài đặt Jenkins
sudo dnf install jenkins -y

# Kích hoạt và khởi động Jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins

# Kiểm tra trạng thái
sudo systemctl status jenkins
```

---

### 1.4 – Truy cập Jenkins Web UI

Mở trình duyệt và truy cập: `http://<EC2-PUBLIC-IP>:8080`

Lấy mật khẩu Admin lần đầu:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Dán mật khẩu vào giao diện **Unlock Jenkins** → chọn **Install suggested plugins** → tạo tài khoản Admin.

---

## Step 2: Tích hợp GitHub với Jenkins

### 2.1 – Cài đặt Git trên EC2

```bash
sudo dnf install git -y

# Xác nhận phiên bản
git --version
```

### 2.2 – Cài đặt Plugin GitHub Integration trên Jenkins

1. Vào **Manage Jenkins** → **Plugins** → tab **Available plugins**.
2. Tìm kiếm `GitHub Integration` → chọn → click **Install**.

### 2.3 – Cấu hình Git trong Jenkins

1. Vào **Manage Jenkins** → **Tools**.
2. Cuộn xuống phần **Git installations** → điền:
   - **Name:** `git`
   - **Path to Git executable:** `git` *(để Jenkins tự tìm)*
3. Click **Save**.

---

## Step 3: Tích hợp Maven với Jenkins

### 3.1 – Cài đặt Maven 3.9.x trên EC2

```bash
cd /opt

sudo wget https://dlcdn.apache.org/maven/maven-3/3.9.15/binaries/apache-maven-3.9.15-bin.tar.gz

sudo tar -xvzf apache-maven-3.9.15-bin.tar.gz

sudo mv apache-maven-3.9.15 maven

sudo rm apache-maven-3.9.15-bin.tar.gz
```

### 3.2 – Thiết lập Environment Variables

Chỉnh sửa file `/etc/profile.d/maven.sh` (cách hiện đại hơn dùng `.bash_profile`):

```bash
sudo tee /etc/profile.d/maven.sh > /dev/null <<'EOF'
export M2_HOME=/opt/maven
export M2=$M2_HOME/bin
export JAVA_HOME=/usr/lib/jvm/java-21-amazon-corretto
export PATH=$M2:$JAVA_HOME/bin:$PATH
EOF

source /etc/profile.d/maven.sh

# Kiểm tra
mvn -version
```

### 3.3 – Cài Plugin và Cấu hình Maven trong Jenkins

1. Vào **Manage Jenkins** → **Plugins** → tìm `Maven Integration` → **Install**.
2. Vào **Manage Jenkins** → **Tools**:
   - **# Cách nhanh nhất – để Java tự tìm path chính xác java - "XshowSettings:all -version 2>&1 | grep "java.home""**
   - **JDK installations:** đặt `JAVA_HOME` = `/usr/lib/jvm/java-21-amazon-corretto`
   - **Maven installations:** đặt `Maven Name` = `maven-3.9.15`
3. Click **Save**.

---

## Step 4: Setup Docker Host trên EC2

### 4.1 – Tạo EC2 Instance mới (Docker Host)

Lặp lại quy trình tạo EC2 như ở Step 1 với tên `docker-host`. Mở thêm port range **8081–9000** trong Security Group.

### 4.2 – Cài đặt Docker Engine

```bash
# Cập nhật hệ thống
sudo dnf update -y

# Cài đặt Docker
sudo dnf install docker -y

# Kích hoạt và khởi động Docker daemon
sudo systemctl enable docker
sudo systemctl start docker

# Kiểm tra phiên bản
docker --version
# Output: Docker version 26.x.x ...
```

### 4.3 – Tạo user `dockeradmin`

```bash
# Tạo user
sudo useradd dockeradmin

# Đặt mật khẩu
sudo passwd dockeradmin

# Thêm vào nhóm docker
sudo usermod -aG docker dockeradmin
```

### 4.4 – Bật xác thực bằng mật khẩu SSH *(chỉ dùng cho lab/demo)*

> 💡 **Production:** Nên dùng SSH key thay vì mật khẩu.

```bash
sudo sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config

# Khởi động lại SSH daemon
sudo systemctl reload sshd
```

### 4.5 – Tạo Dockerfile tùy chỉnh cho Tomcat

```bash
# Tạo thư mục làm việc
sudo mkdir -p /opt/docker
sudo chown -R dockeradmin:dockeradmin /opt/docker
```

Tạo file `/opt/docker/Dockerfile`:

```dockerfile
FROM tomcat:10.1

# Fix issue: webapps trống trong image Tomcat 10+
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps

# Copy WAR artifact từ Jenkins build
COPY ./*.war /usr/local/tomcat/webapps

EXPOSE 8080
```

> ⚠️ **Lưu ý:** Tomcat 10.1 yêu cầu Jakarta EE namespace. Nếu ứng dụng dùng `javax.*`, hãy dùng `tomcat:9.0` thay thế.

### 4.6 – Build và chạy thử container

```bash
# Build image
docker build -t tomcatserver:latest /opt/docker/

# Chạy container thử nghiệm
docker run -d --name tomcat-test -p 8085:8080 tomcatserver:latest

# Kiểm tra
docker ps
```

Truy cập: `http://<DOCKER-HOST-IP>:8085`

---

## Step 5: Tích hợp Docker với Jenkins

### 5.1 – Cài Plugin "Publish Over SSH"

1. Vào **Manage Jenkins** → **Plugins** → **Available plugins**.
2. Tìm `Publish Over SSH` → **Install**.

### 5.2 – Cấu hình Docker Host trong Jenkins

1. Vào **Manage Jenkins** → **System**.
2. Cuộn xuống phần **Publish over SSH** → click **Add** dưới **SSH Servers**:

| Trường | Giá trị |
|---|---|
| **Name** | `docker-host` |
| **Hostname** | Private IP của Docker Host EC2 |
| **Username** | `dockeradmin` |
| **Remote Directory** | `/opt/docker` |

3. Nhập mật khẩu trong **Advanced** → **Password**.
4. Click **Test Configuration** để xác nhận kết nối thành công.
5. **Apply** và **Save**.

---

## Step 6: Tạo Jenkins Job – Build & Copy Artifact sang Docker Host

1. Tạo **New Item** → đặt tên `BuildAndDeploy` → chọn **Maven project** → **OK**.
2. Cấu hình:
   - **Source Code Management:** Git → nhập URL repository GitHub của bạn.
   - **Build Triggers:** chọn `Poll SCM` với schedule `* * * * *` (poll mỗi phút).
   - **Build:** chọn **Root POM** = `pom.xml`, **Goals** = `clean install`.
3. Phần **Post-build Actions** → **Send build artifacts over SSH**:
   - **SSH Server Name:** `docker-host`
   - **Source files:** `webapp/target/*.war`
   - **Remote directory:** `//opt//docker`
4. **Apply** và **Save**.

---

## Step 7: Tự động hóa Build và Deploy toàn diện

### 7.1 – Cập nhật Exec Command trong Jenkins Job

Trong **Post-build Actions** → **Send build artifacts over SSH** → **Exec command**:

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

> 💡 Đoạn script trên xử lý trường hợp container đã tồn tại (tránh lỗi name conflict).

### 7.2 – Kích hoạt Pipeline

Commit bất kỳ thay đổi nào vào GitHub repository. Jenkins sẽ tự động:

1. ✅ Phát hiện thay đổi qua Poll SCM
2. ✅ Clone code từ GitHub
3. ✅ Build `.war` file bằng Maven
4. ✅ Copy artifact sang Docker Host qua SSH
5. ✅ Build Docker image mới
6. ✅ Chạy container mới

### 7.3 – Kiểm tra kết quả

```bash
# Trên Docker Host – kiểm tra container đang chạy
docker ps

# Xem logs của container
docker logs regapp
```

Truy cập ứng dụng: `http://<DOCKER-HOST-IP>:8087/webapp/`

---

## 🧹 Dọn dẹp tài nguyên (Quan trọng!)

Sau khi thực hành xong, nhớ **terminate** các EC2 Instance để tránh phát sinh chi phí:

```bash
# Dừng tất cả container trên Docker Host
docker stop $(docker ps -q)

# Xoá image không dùng
docker image prune -a
```

Sau đó vào **EC2 Console** → chọn các instance → **Instance State** → **Terminate**.

---

## 🔐 Best Practices (Production)

- **SSH Keys** thay vì mật khẩu cho `dockeradmin`.
- Dùng **Jenkins Credentials Store** để lưu secrets (không hardcode trong script).
- Dùng **Jenkinsfile** (Pipeline as Code) thay vì Freestyle Job để version control pipeline.
- Thiết lập **Docker image tagging** theo build number: `regapp:${BUILD_NUMBER}`.
- Cân nhắc dùng **Amazon ECR** thay vì build image trực tiếp trên host.
- Giới hạn Security Group: chỉ mở port cần thiết, không dùng `0.0.0.0/0` cho SSH.

---

## 📌 Tóm tắt Kiến trúc

```
Developer
    │
    │ git push
    ▼
GitHub Repository
    │
    │ Poll SCM (Jenkins)
    ▼
Jenkins Server (EC2 - t3.micro)
  ├── Java 21 (Amazon Corretto)
  ├── Maven 3.9.9
  └── Build .war artifact
    │
    │ Publish Over SSH
    ▼
Docker Host (EC2 - t3.micro)
  ├── Docker Engine 26.x
  ├── Dockerfile (Tomcat 10.1)
  └── Running Container → port 8087
    │
    ▼
http://<docker-host-ip>:8087/webapp/
```

---

## 🛠️ Tác giả & Cộng đồng

Dự án gốc được tạo bởi **[Harshhaa](https://github.com/NotHarshhaa)** 💡.
README này đã được cập nhật lên tech stack hiện đại (2025–2026).

📧 **Kết nối:**

- **GitHub**: [@NotHarshhaa](https://github.com/NotHarshhaa)
- **Blog**: [ProDevOpsGuy](https://blog.prodevopsguytech.com)
- **Telegram**: [Join Here](https://t.me/prodevopsguy)
- **LinkedIn**: [Harshhaa Vardhan Reddy](https://www.linkedin.com/in/harshhaa-vardhan-reddy/)

---

## ⭐ Ủng hộ dự án

Nếu bài hướng dẫn này hữu ích, hãy **star** ⭐ repository và chia sẻ với cộng đồng! 🚀

