# HƯỚNG DẪN CHI TIẾT DEPLOY DỰ ÁN VIETCAR LÊN CPANEL

Tài liệu hướng dẫn quy trình triển khai (deploy) hoàn chỉnh dự án Laravel 10 (VietCar Rental) lên hosting cPanel từ A đến Z, bao gồm chi tiết tạo subdomain, cấu hình `.htaccess`, tối ưu môi trường PHP và xử lý lỗi đặc thù.

---

## 📌 MỤC LỤC
1. [Tạo Subdomain trên cPanel & Cấu hình DNS](#1-tạo-subdomain-trên-cpanel--cấu-hình-dns)
2. [Cấu hình chi tiết file .htaccess](#2-cấu-hình-chi-tiết-file-htaccess)
3. [Chuẩn bị mã nguồn tại máy tính (Local)](#3-chuẩn-bị-mã-nguồn-tại-máy-tính-local)
4. [Truy cập thư mục & Upload mã nguồn trên cPanel](#4-truy-cập-thư-mục--upload-mã-nguồn-trên-cpanel)
5. [Cấu hình PHP & Extensions trên cPanel (Rất quan trọng)](#5-cấu-hình-php--extensions-trên-cpanel-rất-quan-trọng)
6. [Tạo Database & Cấu hình .env](#6-tạo-database--cấu-hình-env)
7. [Phân quyền thư mục & Tạo Storage Symlink](#7-phân-quyền-thư-mục--tạo-storage-symlink)
8. [Checklist xử lý sự cố thường gặp (Troubleshooting)](#8-checklist-xử-lý-sự-cố-thường-gặp)

---

## 1. Tạo Subdomain trên cPanel & Cấu hình DNS

### 1.1. Tạo Subdomain trong cPanel
Tùy thuộc vào giao diện cPanel của nhà cung cấp hosting:

#### Cách 1: Giao diện cPanel mới (Mục Domains)
1. Đăng nhập cPanel -> Tìm mục **Domains** -> Chọn **Create A New Domain**.
2. **Select the type of domain to create**: Chọn `Registered Domain`.
3. **Domain**: Nhập đầy đủ subdomain mong muốn (Ví dụ: `vietcar.trieuthanh.id.vn`).
4. **Document Root**: ⚠️ **BẮT BUỘC BỎ TÍCH** ô *"Share document root (/home/username/public_html) with..."*.
5. Tại ô **Document Root** vừa hiển thị, điền đường dẫn trỏ thẳng vào thư mục `public` của Laravel:
   ```text
   vietcar.trieuthanh.id.vn/public
   ```
   *(Thư mục thực tế trên server sẽ là: `/home/username/vietcar.trieuthanh.id.vn/public`)*.
6. Bấm **Submit**.

#### Cách 2: Giao diện cPanel truyền thống (Mục Subdomains)
1. Tìm mục **Subdomains** trong cPanel.
2. **Subdomain**: Nhập tiền tố (ví dụ: `vietcar`).
3. **Domain**: Chọn tên miền chính từ danh sách (ví dụ: `trieuthanh.id.vn`).
4. **Document Root**: Sửa lại thành `vietcar.trieuthanh.id.vn/public` (hoặc `public_html/vietcar/public`).
5. Bấm **Create**.

---

### 1.2. Cấu hình DNS Record (Nếu quản lý DNS qua Cloudflare hoặc nhà cung cấp ngoài)
Nếu tên miền chính trỏ qua Cloudflare / DNS trung gian, cần thêm 1 bản ghi cho subdomain:
- **Type**: `A`
- **Name**: `vietcar` (hoặc `vietcar.trieuthanh.id.vn`)
- **IPv4 Address**: Nhập địa chỉ IP của hosting cPanel (xem tại mục *Shared IP Address* ở cột bên phải cPanel).
- **Proxy status**: DNS only hoặc Proxied (Cloudflare).

---

## 2. Cấu hình chi tiết file .htaccess

File `.htaccess` quyết định việc định tuyến (routing) của Laravel và bảo mật source code.

### 2.1. File `public/.htaccess` (BẮT BUỘC CÓ)
File này nằm tại thư mục `public/.htaccess`. Nội dung chuẩn:
```apache
<IfModule mod_rewrite.c>
    <IfModule mod_negotiation.c>
        Options -MultiViews -Indexes
    </IfModule>

    RewriteEngine On

    # Bắt buộc chuyển hướng sang HTTPS (Tùy chọn)
    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]

    # Handle Authorization Header
    RewriteCond %{HTTP:Authorization} .
    RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

    # Redirect Trailing Slashes If Not A Folder...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_URI} (.+)/$
    RewriteRule ^ %1 [L,R=301]

    # Send Requests To Front Controller...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^ index.php [L]
</IfModule>
```

---

### 2.2. File `.htaccess` ở thư mục GỐC (Dành cho trường hợp Document Root không trỏ được vào `public`)
Nếu hosting bắt buộc Document Root nằm tại `/home/username/vietcar.trieuthanh.id.vn` (thay vì `/public`), tạo file `.htaccess` ngay tại thư mục gốc ngang hàng với `app`, `bootstrap`:

```apache
<IfModule mod_rewrite.c>
    RewriteEngine On
    
    # Chuyển hướng mọi request vào thư mục public/
    RewriteRule ^(.*)$ public/$1 [L]
</IfModule>

# Chặn người dùng truy cập trực tiếp vào các file nhạy cảm
<Files .env>
    Order allow,deny
    Deny from all
</Files>

<Files composer.json>
    Order allow,deny
    Deny from all
</Files>
```

---

## 3. Chuẩn bị mã nguồn tại máy tính (Local)

Thực hiện tại terminal thư mục dự án `E:\source\vietcar-rental-web`:

### 3.1. Build Frontend Assets (Vite)
```bash
npm run build
```
*(Lệnh này tạo thư mục `public/build/` chứa `manifest.json` và CSS/JS đã được minify)*.

### 3.2. Cài đặt Vendor tương thích PHP 8.2
```bash
composer install --no-dev --optimize-autoloader --no-audit
```

### 3.3. Nén mã nguồn
- Nén thư mục dự án thành file `.zip` (hoặc nén riêng thư mục `vendor` thành `vendor.zip` và `build` thành `build.zip`).
- **Không nén:** `.git/`, `node_modules/`.

---

## 4. Truy cập thư mục & Upload mã nguồn trên cPanel

1. Mở **File Manager** trên cPanel.
2. Bấm vào nút **Settings** (góc trên bên phải) -> Tích chọn **Show Hidden Files (dotfiles)** -> Bấm **Save**.
3. Điều hướng vào thư mục gốc của subdomain:
   ```text
   /home/username/vietcar.trieuthanh.id.vn/
   ```
4. Bấm **Upload** -> Tải file `.zip` mã nguồn lên.
5. Chuột phải vào file `.zip` -> Chọn **Extract**.
6. Đảm bảo cấu trúc file trên hosting như sau:
   ```text
   /home/username/vietcar.trieuthanh.id.vn/
   ├── app/
   ├── bootstrap/
   ├── config/
   ├── database/
   ├── public/
   │   ├── build/              <-- Chứa manifest.json & assets (Vite)
   │   ├── .htaccess           <-- File .htaccess của public
   │   └── index.php
   ├── resources/
   ├── routes/
   ├── storage/
   ├── vendor/                 <-- Thư mục vendor hoàn chỉnh
   ├── .env                    <-- File cấu hình môi trường
   └── composer.json
   ```

---

## 5. Cấu hình PHP & Extensions trên cPanel (Rất quan trọng)

1. Vào mục **Select PHP Version** (hoặc **MultiPHP Manager**).
2. Chọn phiên bản PHP: **PHP 8.2** (hoặc tối thiểu PHP 8.1).
3. Chuyển sang tab **Extensions**:
   - 🔴 **BẮT BUỘC TẮT (BỎ TÍCH):** Extension **`psr`** *(Bật extension C `psr` sẽ gây xung đột cú pháp với Monolog 3 dẫn tới lỗi HTTP 500 trắng trang)*.
   - 🟢 **Bật đầy đủ các extension:** `pdo_mysql`, `fileinfo`, `mbstring`, `openssl`, `tokenizer`, `xml`, `curl`, `bcmath`, `zip`.

---

## 6. Tạo Database & Cấu hình .env

1. **Tạo Database:**
   - Vào **MySQL® Database Wizard**.
   - Tạo Database: `username_vietcar`.
   - Tạo User & Password: `username_dbuser`.
   - Gán quyền: Tích **ALL PRIVILEGES**.
2. **Import dữ liệu:**
   - Vào **phpMyAdmin** -> Chọn Database vừa tạo -> Chọn tab **Import** -> Tải file database `.sql` lên và bấm **Go**.
3. **Cấu hình file `.env`:**
   Mở file `.env` trên cPanel File Manager và cập nhật:
   ```env
   APP_NAME="Vietcar Rental"
   APP_ENV=production
   APP_KEY=base64:nhap_key_cua_ban_tai_day
   APP_DEBUG=false
   APP_URL=https://vietcar.trieuthanh.id.vn

   DB_CONNECTION=mysql
   DB_HOST=127.0.0.1
   DB_PORT=3306
   DB_DATABASE=username_vietcar
   DB_USERNAME=username_dbuser
   DB_PASSWORD=mat_khau_database_user
   ```

---

## 7. Phân quyền thư mục & Tạo Storage Symlink

### 7.1. Phân quyền thư mục ghi (Write Permissions)
Trong File Manager cPanel, click chuột phải -> chọn **Change Permissions**:
- Thư mục `storage/` (và các thư mục con): Đặt quyền **775** (hoặc 755).
- Thư mục `bootstrap/cache/`: Đặt quyền **775** (hoặc 755).

### 7.2. Tạo Storage Symlink (Để hiển thị hình ảnh upload)
- **Nếu cPanel có Terminal:** Chạy lệnh:
  ```bash
  php artisan storage:link
  ```
- **Nếu không có Terminal:** Tạo file `public/symlink.php`:
  ```php
  <?php
  symlink(__DIR__ . '/../storage/app/public', __DIR__ . '/storage');
  echo "Storage linked successfully!";
  ```
  Truy cập `https://vietcar.trieuthanh.id.vn/symlink.php` 1 lần trên trình duyệt rồi xóa file này đi.

---

## 8. Checklist xử lý sự cố thường gặp

| Hiện tượng / Lỗi | Nguyên nhân | Cách xử lý |
| :--- | :--- | :--- |
| **HTTP ERROR 500 (Trắng trang)** | `.env` cấu hình sai DB, thiếu `APP_KEY`, hoặc dính cache cũ. | Đổi tạm `APP_DEBUG=true` trong `.env` để xem lỗi cụ thể. Xóa file trong `bootstrap/cache/`. |
| **Declaration of Monolog\Logger must be compatible with PsrExt...** | cPanel đang bật PHP extension `psr`. | Vào **Select PHP Version** -> Bỏ tích (tắt) extension **`psr`**. |
| **Vite manifest not found at: .../public/build/manifest.json** | Chưa upload thư mục `public/build/`. | Chạy `npm run build` ở local, nén thư mục `build/` upload vào `public/`. |
| **Composer detected issues: requires PHP >= 8.4.0** | `composer.lock` tạo trên PHP 8.4. | Thêm `"config": {"platform": {"php": "8.2.0"}, "platform-check": false}` vào `composer.json`. |
| **The Process class relies on proc_open, which is not available** | cPanel chặn hàm `proc_open`. | Cài `composer install` ở máy local rồi nén thư mục `vendor` upload lên cPanel. |
| **404 Not Found khi truy cập các route con** | Thiếu file `public/.htaccess`. | Tạo file `public/.htaccess` với nội dung rewrite front controller chuẩn của Laravel. |
