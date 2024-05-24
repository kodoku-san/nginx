# Load Balancing với Nginx

Trong môi trường phân phối ứng dụng, load balancing là một kỹ thuật quan trọng giúp phân phối tải giữa các server backend để cải thiện hiệu suất. Nginx là một trong những máy chủ web phổ biến nhất, cung cấp một số tính năng mạnh mẽ để cấu hình load balancing.

Khi một yêu cầu HTTP được gửi đến máy chủ load balancer, nó sẽ chuyển hướng yêu cầu đó đến một trong các server backend theo một thuật toán phân phối tải nhất định.


### 1. Cài đặt Nginx

Đầu tiên, bạn cần cài đặt Nginx. Trên các hệ thống dựa trên Debian/Ubuntu, bạn có thể cài đặt bằng cách chạy:

```bash
sudo apt update
sudo apt install nginx
```

#### Tạo các thư mục cho các phiên bản ứng dụng

1. **Tạo các thư mục** cho các phiên bản ứng dụng khác nhau:

    ```sh
    sudo mkdir -p /var/www/html/app1
    sudo mkdir -p /var/www/html/app2
    sudo mkdir -p /var/www/html/app3
    sudo mkdir -p /var/www/html/app4
    ```

2. **Thêm nội dung đơn giản** vào các thư mục này để kiểm tra:

    ```sh
    echo "App 1" | sudo tee /var/www/html/app1/index.html
    echo "App 2" | sudo tee /var/www/html/app2/index.html
    echo "App 3" | sudo tee /var/www/html/app3/index.html
    echo "App 4" | sudo tee /var/www/html/app4/index.html
    ```

---
### 2. Tạo cấu hình load balancing

Trong hướng dẫn này, sẽ dùng cấu hình sites mặc định từ `sites-enabled/default` để cấu hình load balancing. 
Mở tệp cấu hình mặc định của Nginx:

```bash
sudo nano /etc/nginx/sites-enabled/default
```

Xóa nội dung cũ đi. Sau đó thêm nội dung sau vào tệp cấu hình:

```nginx
upstream backend {
#   ip_hash;
#   least_conn;
    server 127.0.0.1:8081 weight=1;
    server 127.0.0.1:8082 weight=1;
    server 127.0.0.1:8083 weight=1;
    server 127.0.0.1:8084 weight=1;
#        sticky cookie srv_id expires=1h path=/;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
    }
}
```

- `upstream {backend}`: Khai báo các server {backend}(đặt tên gì cũng được). 
- `server`: Cấu hình server Nginx, lắng nghe trên port 80.
  - `location`: Cấu hình định tuyến, chuyển hướng các yêu cầu đến server backend.
  - Trong upstream, có thể sử dụng các thuật toán phân phối tải như `ip_hash`, `least_conn`, `sticky cookie`. Cụ thể ở phần dưới mình sẽ giới thiệu sơ về chức năng cũng như thuật toán nó.
- Ngoài ra đây là demo load balancing cho 4 server backend chạy trên cùng một máy chủ, do đó sử dụng 127.0.0.1 (localhost) làm địa chỉ IP. Thực tế, cần phải thay đổi thành địa chỉ IP của các server backend.

---
Tạo một tệp mới trong thư mục `/etc/nginx/conf.d/` để cấu hình các server backend, ví dụ: `backend.conf`.

```bash
sudo nano /etc/nginx/conf.d/backend.conf
```

```nginx
server {

    listen 8081;
    index index.html;
    add_header Custom-Header "App 1";

    location / {
        # đường dẫn đến folder app
       root /var/www/html/app1;
    }
}
server {

    listen 8082;
    index index.html;
    add_header Custom-Header "App 2";

    location / {
        # đường dẫn đến folder app
       root /var/www/html/app2;
    }
}
server {

    listen 8083;
    index index.html;
    add_header Custom-Header "App 3";

    location / {
        # đường dẫn đến folder app
       root /var/www/html/app3;
    }
}
server {

    listen 8084;
    index index.html;
    add_header Custom-Header "App 4";

    location / {
        # đường dẫn đến folder app
       root /var/www/html/app4;
    }
}
```

- Trong `backend.conf` chúng ta sẽ cấu hình các server backend, mỗi server sẽ lắng nghe trên một port khác nhau. 
  - `add_header`: Thêm header vào response của server.
  - `location`: Cấu hình đường dẫn đến folder chứa ứng dụng.
- Tùy vào dự án của mình, bạn cần thay đổi đường dẫn `/var/www/html/app1`, `/var/www/html/app2`, `/var/www/html/app3` , `/var/www/html/app4` tương ứng với ứng dụng.
---
### 3. Mở port cho server backend

Sử dụng ufw để mở port `80` và `8081-8084`.

Cài đặt ufw:

```bash
sudo apt install ufw
sudo ufw enable
```

Mở port:

```bash
sudo ufw allow 80
sudo ufw allow 8081
sudo ufw allow 8082
sudo ufw allow 8083
sudo ufw allow 8084
```

Kiểm tra trạng thái và list port:

```bash
sudo ufw status
```
---
### 4. Kích hoạt cấu hình và khởi động lại Nginx

```bash
sudo nginx -t
sudo systemctl restart nginx
```
- Hãy nhớ khởi động lại `Nginx` sau khi thay đổi cấu hình để áp dụng các thay đổi.
---
### 5. Kiểm tra

Kiểm tra kết nối trực tiếp đến các cổng đã cấu hình để đảm bảo rằng chúng đang hoạt động. Truy cập từ trình duyệt hoặc sử dụng curl:

```sh
curl http://localhost:8081
curl http://localhost:8082
curl http://localhost:8083
curl http://localhost:8084
```
- Nếu các cổng này không trả về kết quả mong muốn, hãy kiểm tra lại cấu hình Nginx và đảm bảo rằng các dịch vụ đang chạy.
---
Mở trình duyệt và truy cập địa chỉ IP của VPS (hoặc tên miền nếu có):

```sh
http://your_vps_ip
```
- Xác nhận rằng Nginx đang phân phối request giữa các phiên bản ứng dụng chạy trên các cổng 8081, 8082, 8083 và 8084. Bạn sẽ thấy nội dung "App 1", "App 2", "App 3" và "App 4" xen kẽ khi tải lại trang.
---
Chạy lệnh sau để get tuần tự qua các server backend:

```sh
while true; do curl -sI <load-balancer-ip:port> | tr -d '\r' | sed -En 's/^Custom-Header: (.*)/\1/p'; sleep 1; done
```
- Lệnh trên sẽ hiển thị header `Custom-Header` của các server backend mà load balancer đang chuyển hướng đến.
---
### 6. Các Tùy Chọn Cấu Hình `upstream`

Trong cấu hình `upstream` của Nginx, bạn có thể thêm một số tùy chọn để điều chỉnh cách thức phân phối tải và quản lý các server backend. Dưới đây là một số tùy chọn phổ biến mà bạn có thể sử dụng trong block `upstream`:

0. **Round Robin** (mặc định):

   - Thuật toán phân phối tải mặc định của Nginx là `round-robin`, nghĩa là mỗi yêu cầu sẽ được chuyển đến server tiếp theo trong danh sách một cách tuần tự.

    ```nginx
    upstream backend {
        server 127.0.0.1:8081;
        server 127.0.0.1:8082;
        server 127.0.0.1:8083;
        server 127.0.0.1:8084;
    }
    ```

1. **Weight (Trọng số)**:
   - Tăng trọng số cho các server mạnh hơn để chúng nhận được nhiều yêu cầu hơn.
   
   ```nginx
    upstream backend {
        server 127.0.0.1:8081 weight=3;
        server 127.0.0.1:8082 weight=1;
        server 127.0.0.1:8083 weight=2;
        server 127.0.0.1:8084 weight=1;
    }
   ```

2. **Max Fails và Fail Timeout**:
   - Giới hạn số lần thử thất bại trước khi đánh dấu server là không khả dụng và định nghĩa thời gian chờ trước khi thử lại.

   ```nginx
    upstream backend {
        server 127.0.0.1:8081 max_fails=3 fail_timeout=30s;
        server 127.0.0.1:8082 max_fails=3 fail_timeout=30s;
        server 127.0.0.1:8083 max_fails=3 fail_timeout=30s;
        server 127.0.0.1:8084 max_fails=3 fail_timeout=30s;
    }
   ```

3. **Backup**:
   - Định nghĩa server backup chỉ được sử dụng khi tất cả các server chính không khả dụng.
   - `8084`: chắc hẳn tuyệt vọng lắm anh mới tìm đến tôi <(")

   ```nginx
    upstream backend {
        server 127.0.0.1:8081;
        server 127.0.0.1:8082;
        server 127.0.0.1:8083;
        server 127.0.0.1:8084 backup;
    }
   ```

4. **Keepalive**:

- Keepalive là một kỹ thuật giữ kết nối mạng mở giữa client và server để giảm thời gian cần thiết để thiết lập kết nối TCP mới. Khi một client gửi yêu cầu HTTP đến server, server sẽ giữ kết nối mở sau khi phản hồi yêu cầu của client, khi client gửi yêu cầu tiếp theo, server sẽ sử dụng kết nối đã mở để phản hồi yêu cầu đó. Điều này giúp giảm thời gian cần thiết để thiết lập kết nối TCP mới và cải thiện hiệu suất của ứng dụng.
   - Duy trì kết nối lâu dài với các server backend để cải thiện hiệu suất.

   ```nginx
    upstream backend {
        server 127.0.0.1:8081;
        server 127.0.0.1:8082;
        server 127.0.0.1:8083;
        server 127.0.0.1:8084;
        keepalive 16;
    }
   ```

5. **Least Connections**:
   - Sử dụng thuật toán phân phối tải dựa trên số kết nối ít nhất.
   - Thuật toán này là một trong những cách phân phối tải phổ biến nhất, nó chuyển request đến server có ít kết nối nhất. 
   - Nếu có nhiều server có số kết nối nhỏ nhất, request sẽ được chuyển đến server đầu tiên trong danh sách.

   ```nginx
    upstream backend {
        least_conn;
        server 127.0.0.1:8081;
        server 127.0.0.1:8082;
        server 127.0.0.1:8083;
        server 127.0.0.1:8084;
    }
   ```

6. **IP Hash**:
   - Sử dụng thuật toán phân phối tải dựa trên địa chỉ IP của client để đảm bảo rằng cùng một client luôn được chuyển đến cùng một server.
   - Nói dễ hiểu là một client sẽ luôn được chuyển đến một server trong một khoảng thời gian nhất định.

   ```nginx
    upstream backend {
        ip_hash;
        server 127.0.0.1:8081;
        server 127.0.0.1:8082;
        server 127.0.0.1:8083;
        server 127.0.0.1:8084;
    }
   ```
7. **Sticky Cookie**:
   - Sử dụng thuật toán phân phối tải dựa trên cookie để đảm bảo rằng cùng một client luôn được chuyển đến cùng một server.
   - Cookie sẽ được tạo ra và gửi đến client, sau đó client sẽ gửi cookie này trong mỗi yêu cầu tiếp theo.

   ```nginx
    upstream backend {
        sticky cookie srv_id expires=1h path=/;
        server 127.0.0.1:8081;
        server 127.0.0.1:8082;
        server 127.0.0.1:8083;
        server 127.0.0.1:8084;
    }
    ``` 

Dưới đây là một ví dụ cấu hình `upstream` tích hợp nhiều tùy chọn:

```nginx
upstream backend {
    least_conn;
    server 127.0.0.1:8081 weight=3 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8082 weight=1 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8083 weight=2 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8084 weight=2 max_fails=3 fail_timeout=30s;
    keepalive 16;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
    }
}
```

- Trong ví dụ này, chúng ta sử dụng `least_conn` để phân phối tải dựa trên số kết nối ít nhất, cùng với các tùy chọn `weight`, `max_fails`, `fail_timeout`, `keepalive`. Nhằm cải thiện hiệu suất của hệ thống.
---
Sau khi cập nhật cấu hình `nginx.conf`, đừng quên kiểm tra cú pháp và khởi động lại Nginx:

```sh
sudo nginx -t
sudo systemctl restart nginx
```

### Tài Liệu Tham Khảo

- **Nginx Upstream Module**: [Nginx Documentation](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)

### Kết Luận

Trên đây là hướng dẫn cấu hình một hệ thống load balancing đơn giản bằng Nginx. Bằng cách sử dụng `upstream` và `server` block, bạn có thể phân phối tải giữa các server backend và cải thiện hiệu suất của ứng dụng của mình. 
Bên cạnh đó là cách sử dụng một số tùy chọn cấu hình phổ biến để tùy chỉnh cách thức hoạt động của hệ thống load balancing.

Tuy nhiên hãy nhớ rằng, cấu hình load balancing chỉ là một phần của việc tăng cường hiệu suất hệ thống. Nên xem xét các yếu tố khác như cache, tối ưu hóa cơ sở dữ liệu, tối ưu hóa ứng dụng, và nhiều yếu tố khác để đảm bảo rằng hệ thống của bạn hoạt động ổn định và hiệu quả. Chúc các bạn thành công!

---
`KODOKU` - [Github⭐](https://github.com/kodoku-san)
