# 📘 Elearn Backend – Production Deployment Guide

This document explains how to build, configure, deploy, and run the **Elearn Backend API** on an **Azure Linux VM** with an **Azure MySQL Flexible Server (Private)** using a production‑ready setup.

The backend is built using **.NET 8 + Entity Framework + MySQL** and exposes APIs to manage courses and lessons.

---

## 🚀 1️⃣ Architecture Overview

This backend follows a secure and production‑grade architecture:

* **Azure Linux VM** → Hosts .NET 8 Kestrel API
* **Azure MySQL Flexible Server** (Private access, delegated subnet)
* **Private DNS Zone** for MySQL name resolution
* **GitHub Actions Pipeline** → Builds + deploys to VM via SSH
* **Systemd service** → Auto‑start, auto‑restart, runs API continuously
* **Kestrel** → Listens on `http://0.0.0.0:5000`
* API exposed through VM Public IP or frontend VM

---

## ⚙️ 2️⃣ Prerequisites for Developers

### **Local Tools Required**

* .NET SDK 8.0+
* MySQL Server or Azure MySQL
* Git
* Postman
* Docker (optional)

### **Azure Requirements**

* Azure Linux VM (Ubuntu)
* Azure MySQL Flexible Server
* Private DNS Zone
* Delegated Subnet
* SSH Private Key
* MySQL username/password

---

## 📁 3️⃣ Clone the Repository

```bash
git clone https://github.com/your-repo/elearn-backend.git
cd elearn-backend
```

---

## 🗄️ 4️⃣ Database Setup

The application expects a database named **elearn**.

### ✔ Create database manually

```sql
CREATE DATABASE elearn;
```

### ✔ Create required table schema

```sql
CREATE TABLE IF NOT EXISTS Courses (
    CourseId INT AUTO_INCREMENT PRIMARY KEY,
    CourseName VARCHAR(255) NOT NULL,
    InstructorName VARCHAR(255) NOT NULL,
    Lessons JSON NOT NULL
);
```

### ✔ Terraform users

```hcl
mysql_databases = {
  elearn = {
    database_name = "elearn"
    server_key    = "main"
  }
}
```

---

## 🔧 5️⃣ Local Development Setup

Edit **appsettings.json**:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Port=3306;Database=elearn;User=root;Password=yourpassword;"
  }
}
```

Install dependencies:

```bash
dotnet restore
```

Build:

```bash
dotnet build --configuration Release
```

Run locally:

```bash
dotnet run
```

---

## 🏗️ 6️⃣ Build & Publish (Local)

```bash
dotnet publish -c Release -o ./publish
```

This generates:

```
publish/ElearnBackend.dll
```

Run:

```bash
cd publish
dotnet ElearnBackend.dll
```

---

## 🐳 7️⃣ Docker (Optional Local)

Build:

```bash
docker build -t elearn-backend .
```

Run:

```bash
docker run -p 5000:5000 -p 5001:5001 elearn-backend
```

---

## 🖥️ 8️⃣ Azure VM Production Deployment

### **8.1 Install .NET 8 Runtime**

```bash
sudo apt update
sudo apt install -y dotnet-runtime-8.0
sudo apt install -y aspnetcore-runtime-8.0
```

Verify:

```bash
dotnet --list-runtimes
```

### **8.2 Create Deployment Directory**

```bash
sudo mkdir -p /var/www/elearn-backend
sudo chown -R <vmusername>:<vmusername> /var/www/elearn-backend
```

### **8.3 Set Environment Variables**

```bash
sudo nano /etc/environment
```

Add:

```
ASPNETCORE_ENVIRONMENT=Production
ConnectionStrings__DefaultConnection="Server=<servername>.mysql.database.azure.com;Port=3306;Database=elearn;User Id=<username>;Password=<password>;SslMode=Required;"
```

Apply:

```bash
source /etc/environment
```

### **8.4 Create systemd Service**

```bash
sudo nano /etc/systemd/system/elearn-backend.service
```

Paste:

```ini
[Unit]
Description=Elearn Backend API
After=network.target

[Service]
WorkingDirectory=/var/www/elearn-backend
ExecStart=/usr/bin/dotnet /var/www/elearn-backend/ElearnBackend.dll
Restart=always
RestartSec=10
User=<vmusername>
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=ASPNETCORE_URLS=http://0.0.0.0:5000
Environment="ConnectionStrings__DefaultConnection=Server=<servername>.mysql.database.azure.com;Port=3306;Database=elearn;User Id=<username>;Password=<password>;SslMode=Required;"

[Install]
WantedBy=multi-user.target
```

Reload + start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable elearn-backend
sudo systemctl restart elearn-backend
```

Check logs:

```bash
sudo journalctl -u elearn-backend -n 50 --no-pager
```

---

## 🚚 9️⃣ GitHub Actions Deployment Pipeline

### Required Secrets

| Secret    | Example         |
| --------- | --------------- |
| `VM_HOST` | 20.xx.xx.xx     |
| `VM_USER` | azureuser       |
| `VM_KEY`  | private SSH key |

### **GitHub Workflow: `.github/workflows/deploy.yml`**

```yaml
name: Deploy Elearn Backend

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'

    - name: Publish project
      run: dotnet publish -c Release -o publish

    - name: Deploy to VM
      uses: appleboy/scp-action@v0.1.7
      with:
        host: ${{ secrets.VM_HOST }}
        username: ${{ secrets.VM_USER }}
        key: ${{ secrets.VM_KEY }}
        source: "publish/*"
        target: "/var/www/elearn-backend"

    - name: Restart service
      uses: appleboy/ssh-action@v0.1.7
      with:
        host: ${{ secrets.VM_HOST }}
        username: ${{ secrets.VM_USER }}
        key: ${{ secrets.VM_KEY }}
        script: |
          sudo systemctl daemon-reload
          sudo systemctl restart elearn-backend
```

---

## 📡 🔟 Testing the API

### **From VM**

```bash
curl http://localhost:5000/api/Course
```

### **From outside**

```
http://<VM_PUBLIC_IP>:5000/api/Course
```

Swagger UI:

```
http://<VM_PUBLIC_IP>:5000/swagger
```

---

## 📚 1️⃣1️⃣ API Endpoints

| Method | Endpoint         | Description      |
| ------ | ---------------- | ---------------- |
| GET    | /api/Course      | Get all courses  |
| POST   | /api/Course      | Add new course   |
| GET    | /api/Course/{id} | Get course by ID |
| DELETE | /api/Course/{id} | Delete course    |

---

## 🎯 Optional Enhancements

* Add NGINX reverse proxy (`https://api.domain.com`)
* Add SSL using Certbot
* Add fail2ban + firewall rules
* Auto‑migrations on startup
* Add load balancer + autoscaling
* Add monitoring (Azure Monitor + Serilog)

---

**✔ This README serves as your complete reference for development and production deployment of the Elearn Backend API.**
