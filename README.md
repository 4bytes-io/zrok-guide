# Hướng dẫn cài đặt và câu hình cho zrok tunnel

## Tổng quan

Để zrok có thể tunnel từ 1 server của user đến public domain, zrok cần cài các component sau:

1. Hệ thống network của openziti trên server của mình bao gồm:
  - openziti controller
  - openziti router

2. zrok controller và frontend trên hệ thống server của mình

3. zrok trên máy của user

ngoài ra cần có 1 wildcard dns trỏ vào server của zrok controller

## Cài Openziti

Cài openziti controller dựa trên hướng dẫn [này](https://openziti.io/docs/category/deployments).

Openziti có thể cài trực tiếp trên linux server, bằng docker hoặc trên kubernetes qua helm.

Nếu cài ziti bằng docker hoặc kubernetes thì phải cải cli qua [link](https://openziti.io/docs/downloads/).

Sau khi cài openziti controller chạy command sau để thêm router vào ziti controller

```sh
ziti edge login <controller host>:<controller port> -u <admin> -p <password>
```
&lt;admin&gt; và &lt;password&gt; là tài khoản admin khai báo trong lúc cài controller
(biến môi trường ZITI_USER và ZITI_PWD, hoặc giá trị khai báo nếu cài trực tiếp trên linux)

```sh
ziti edge create edge-router <router name> -o router.jwt
```
command trên sẽ tạo ra 1 file chứa enroll token, token này sẽ được dùng trong lúc cài ziti router

Sau đó cài ziti router theo hướng dẫn trong [link ở trên](https://openziti.io/docs/category/deployments)

chạy command sau để kiểm tra việc cài openziti đã thành công:
```sh
ziti edge list edge-routers
```

## Cài zrok controller và frontend

Cài zrok theo hướng dẫn trên [link](https://docs.zrok.io/docs/category/self-hosting/)

Bản guide này sẽ chỉ hướng dẫn việc cài trên Linux server
Cài cli của zrok dựa trên tài liệu [này](https://docs.zrok.io/docs/guides/install/linux/)

Zrok cần được cài trên cả server controller và frontend (có thể chung server cũng được)

### Zrok controller

Sau khi cài zrok trên server controller, tạo 1 file với tên `ctrl-conf.yaml` với nội dung như trong [file này](./ctrl-conf.yaml).
Set biến môi trường `ZROK_ADMIN_TOKEN` với token trong file `ctrl-conf.yaml`. 
Có thể tạo 1 token random bằng command như sau: 
```sh
LC_ALL=C tr -dc _A-Z-a-z-0-9 < /dev/urandom | head -c32
```

Chạy command sau:
```sh
zrok admin bootstrap ctrl-conf.yaml
```

Sau đó chạy zrok controller process: 
```sh
zrok controller ctrl-conf.yaml
```

Tạo 1 zrok frontend
```sh
zrok admin create frontend <id> <frontend name> http://{token}.<dns>
```
&lt;id&gt; có thể lấy bằng cách chạy command sau trên server của openziti:
```sh
ziti edge list identities
```
id là giá trị ID của dòng có Name là public
&lt;dns&gt; là phần không thay đổi của wildcard dns, VD: nếu wildcard dns là *.zrok.fcs.ninja thì giá trị ở đây là zrok.fcs.ninja

- **lưu ý** 
    1. `{token}` là 1 string fixed, không phải thay giá trị gì vào đây hết.
    2. nếu cmd `list identities` báo `UNAUTHORIZED` thì chạy `ziti edge login` như ở trên trước

### Zrok frontend

Đầu tiên cần trỏ wildcard dns vào ip của zrok frontend.
Trên server frontend tạo file `fe-conf.yaml` như sau:
```yaml
v: 3
host_match: <dns> # nếu wildcard dns là *.zrok.fcs.ninja thì giá trị ở đây là zrok.fcs.ninja
address: 0.0.0.0:<fe port>
```

chạy frontend server bằng command:
```sh
zrok access <fe name> fe-conf.yaml
```

### Tạo zrok user

Trên 1 máy tính bất kỳ đã cài zrok, set 2 biến môi trường sau:
- Set biến môi trường `ZROK_ADMIN_TOKEN` với token trong file `ctrl-conf.yaml`. 
- Set biến môi trường `ZROK_API_ENDPOINT` với giá trị *&lt;controller url&gt;:&lt;controller port&gt;*.

Chạy command sau để thêm zrok user:
```sh
zrok admin create account <account name> <password> 
```
Cmd trên sẽ trả về 1 giá trị token, giá trị này sử dụng để tunnel

Account trên có thể sử dụng để đăng nhập vào zrok controller.

## Cài zrok trên máy user và tunneling

Trên server user sau khi đã cài zrok, chạy command sau:

```sh
zrok enable <user token>
```
Token ở đây là giá trị đầu ra khi tạo user. Nếu chưa lưu lại có thể dùng zrok user đăng nhập vào trang của controller để xem lại.

Tunnel bằng command sau:
```sh
zrok share public localhost:<port to tunnel>
```
trong command output sẽ có trả về dns để truy cập vào port được tunnel qua frontend
