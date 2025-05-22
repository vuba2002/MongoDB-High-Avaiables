# 🚀 Triển Khai MongoDB Replica Set Kèm HA Với Keepalived

## 🧱 Mô Hình Tổng Quan

- Hệ thống gồm **3 servers Ubuntu**
- Cài đặt MongoDB 7.0 và cấu hình **Replica Set**
- Cấu hình **Keepalived** để cấp địa chỉ **VIP (Virtual IP)** giúp đảm bảo tính sẵn sàng cao (High Availability)

---

## 📌 Bước 1: Cài Đặt MongoDB Trên Cả 3 Servers

### Cài đặt công cụ hỗ trợ
```bash
apt install -y net-tools telnet traceroute
```

### Thêm kho MongoDB 7.0 và cài đặt
```bash
apt install -y software-properties-common gnupg apt-transport-https ca-certificates
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-7.0.list
apt update
apt install -y mongodb-org
```

---

## ⚙️ Bước 2: Cấu Hình Replica Set

### Sửa file `/etc/mongod.conf` trên cả 3 servers
```yaml
net:
  bindIp: 0.0.0.0

replication:
  replSetName: "mongodb-cluster"
```

### Khởi động lại dịch vụ MongoDB
```bash
systemctl restart mongod
```

---

## 🧠 Bước 3: Khởi Tạo Replica Set (Trên server 1)

### Truy cập MongoDB shell
```bash
mongosh
```

### Khởi tạo cluster
```javascript
rs.initiate({
  _id: "mongodb-cluster",
  members: [
    { _id: 0, host: "database-1:27017", priority: 2 },
    { _id: 1, host: "database-2:27017", priority: 1 },
    { _id: 2, host: "database-3:27017", priority: 0.5 }
  ]
})
```

> ✅ **Lưu ý:** Node có `priority` cao hơn sẽ được ưu tiên làm **Primary**

### Kiểm tra trạng thái
```javascript
rs.status()
rs.isMaster()
```

---

## 🔐 Bước 4: Bật Chế Độ Bảo Mật MongoDB Với `authorization` + `keyFile`

### 1. Tạo `keyfile` bí mật trên một server (ví dụ server 1)
```bash
openssl rand -base64 756 > /etc/mongod.keyfile
chmod 400 /etc/mongod.keyfile
chown mongodb:mongodb /etc/mongod.keyfile
```

### 2. Sao chép keyfile này sang 2 server còn lại
```bash
scp /etc/mongod.keyfile user@worker-node-1:/etc/mongod.keyfile
scp /etc/mongod.keyfile user@worker-node-2:/etc/mongod.keyfile
```

> Trên các node nhận:
```bash
chmod 400 /etc/mongod.keyfile
chown mongodb:mongodb /etc/mongod.keyfile
```

### 3. Tạo user `admin` (thực hiện trên node **Primary**):
```bash
mongosh
```

```javascript
use admin
db.createUser({
  user: "admin",
  pwd: "your-secure-password",
  roles: [ { role: "root", db: "admin" } ]
})
```

### 4. Sửa `/etc/mongod.conf` trên cả 3 servers:
```yaml
security:
  authorization: enabled
  keyFile: /etc/mongod.keyfile
```

### 5. Khởi động lại MongoDB
```bash
systemctl restart mongod
```


> Sau đó dùng: `mongosh --username admin -p` để truy cập

---

## 🧪 Bước 5: Kiểm Tra Đồng Bộ Dữ Liệu (Server 2 hoặc 3)

```bash
mongosh --username admin -p
```

```javascript
show dbs
use devopseduvndb
show collections
```

> Thử chèn dữ liệu trên secondary sẽ thất bại:
```javascript
db.accounts.insertOne({ "username": ["elroydevops", "manhnv"] })
```

---

## 💾 Bước 6: Thử Tính Năng HA

### Trên server 1 (primary)
```javascript
use devopseduvndb
db.courses.insertOne({ "name": ["DevOps for freshers", "pipeline DevSecOps", "HA tools"] })
```

### Sau đó khởi động lại server 1 để mô phỏng lỗi
```bash
reboot
```

### Trên server 2 hoặc 3
```bash
mongosh --username admin -p
rs.status()
```

---

## 🌐 Bước 7: Cấu Hình Keepalived (VIP)

### Cài đặt trên tất cả các server
```bash
apt install -y keepalived
```

### File cấu hình `/etc/keepalived/keepalived.conf`

#### Trên server 1 (MASTER)
```conf
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass password
    }
    virtual_ipaddress {
        192.168.154.140
    }
}
```

#### Trên server 2 (BACKUP)
```conf
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass password
    }
    virtual_ipaddress {
        192.168.154.140
    }
}
```

#### Trên server 3 (BACKUP)
```conf
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass password
    }
    virtual_ipaddress {
        192.168.154.140
    }
}
```

> 🔎 **Lưu ý:** Thay `ens33` bằng tên card mạng thực tế (`ip a` để xem).

### Khởi động lại dịch vụ
```bash
systemctl restart keepalived
```

---

## ✅ Bước 8: Kiểm Tra VIP Hoạt Động

- Ping `192.168.154.140` từ các máy khác
- Tắt server đang giữ VIP, kiểm tra VIP chuyển sang máy khác chưa
- Truy cập MongoDB qua địa chỉ VIP: `mongosh --host 192.168.213.14:27017 --username admin -p`

---

## 🎯 Tổng Kết

Bạn đã hoàn tất cài đặt một cụm MongoDB Replica Set có tính năng High Availability sử dụng Keepalived và bảo mật nội bộ với `keyFile + authorization`.

> ✅ **Khuyến nghị:** Kết hợp thêm công cụ giám sát như `Prometheus + Grafana` hoặc `MongoDB Compass` để quan sát dữ liệu rõ hơn.
