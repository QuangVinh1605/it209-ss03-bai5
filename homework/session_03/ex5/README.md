# Bài 5 – Thiết lập thư mục Web an toàn tại `/opt/`

## 1. Mục tiêu

Cô lập mã nguồn website tĩnh và log Nginx tại `/opt/my-app`, đồng thời phân quyền giữa user quản trị/phát triển `devops` và user chạy Nginx `www-data`.

## 2. Cấu trúc thư mục

```text
/opt/my-app/
├── html/
│   └── index.html
└── logs/
    ├── access.log
    └── error.log
```

## 3. User/group

```text
devops
www-data
```

Worker processes của Nginx chạy bằng `www-data`.

## 4. Ownership

Output thực tế:

```text
$ ls -la /opt/my-app/
total 16
drwxr-xr-x  4 devops www-data 4096 Oct  1 23:01 .
drwxr-xr-x 10 root   root     4096 Oct  1 23:01 ..
drwxr-xr-x  2 devops www-data 4096 Oct  1 23:01 html
drwxrwxr-x  2 devops www-data 4096 Oct  1 23:01 logs

$ ls -la /opt/my-app/html/
total 12
drwxr-xr-x 2 devops www-data 4096 Oct  1 23:01 .
drwxr-xr-x 4 devops www-data 4096 Oct  1 23:01 ..
-rw-r--r-- 1 devops www-data  203 Oct  1 23:04 index.html

$ ls -la /opt/my-app/logs/
total 12
drwxrwxr-x 2 devops www-data 4096 Oct  1 23:01 .
drwxr-xr-x 4 devops www-data 4096 Oct  1 23:01 ..
-rw-r--r-- 1 root   root      172 Oct  1 23:04 access.log
-rw-r--r-- 1 root   root        0 Oct  1 23:01 error.log
```

Sau khi Nginx tạo log, các file log có owner `root:root` trên máy này; quyền đọc của Nginx và việc ghi log vẫn hoạt động bình thường.

## 5. Permission

```text
/opt/my-app                 755
/opt/my-app/html            755
/opt/my-app/html/index.html 644
/opt/my-app/logs            775
```

`devops` sở hữu mã nguồn và ghi được `index.html`; `www-data` có thể traverse/đọc website và ghi trong thư mục `logs`.

## 6. Nginx configuration

```nginx
root /opt/my-app/html;

access_log /opt/my-app/logs/access.log;
error_log /opt/my-app/logs/error.log;
```

Server Block dùng `server_name my-app.local` và được enable qua `sites-enabled/my-app`. Server Block `ptit-app` vẫn được giữ nguyên.

## 7. Nginx test

Output thực tế:

```text
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

$ systemctl is-active nginx
active
```

## 8. Website test

Output thực tế (rút gọn phần progress của curl):

```text
$ curl -i -H "Host: my-app.local" http://127.0.0.1
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Content-Type: text/html
Content-Length: 203

<h1>My App is running!</h1>
<p>Production Web Server</p>
```

## 9. Devops write test

Thực hiện bằng user `devops`, không sử dụng `sudo`:

```bash
echo "Update Test" >> /opt/my-app/html/index.html
```

Kết quả thực tế:

```text
</html>
Update Test
Update Test
```

Hai dòng `Update Test` xuất hiện vì lệnh test đã được chạy hai lần trong quá trình thao tác; điều này xác nhận `devops` có quyền ghi trực tiếp vào HTML.

## 10. Access log

Output thực tế:

```text
$ tail /opt/my-app/logs/access.log
127.0.0.1 - - [01/Oct/2026:23:02:04 +0700] "GET / HTTP/1.1" 200 179 "-" "curl/8.18.0"
127.0.0.1 - - [01/Oct/2026:23:04:59 +0700] "GET / HTTP/1.1" 200 203 "-" "curl/8.18.0"
```

`error.log` đã được Nginx tạo thành công và hiện chưa có lỗi để ghi nhận.

## 11. Kết luận

- Website nằm tại `/opt/my-app/html/`.
- Logs nằm tại `/opt/my-app/logs/`.
- Ownership và quyền là phù hợp với mục tiêu bài tập, không dùng `777`.
- `devops` ghi được HTML không cần sudo.
- Nginx đọc được HTML, ghi access/error log, và trả HTTP `200`.
- `ptit-app` vẫn còn enable tại `/etc/nginx/sites-enabled/ptit-app`.
