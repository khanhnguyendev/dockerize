# Dockerfile: Optimize build time and image size

## Tại sao phải Omtimize Dockerfile
- Tối ưu chi phí lưu trữ (container registry)
- Tối ưu thời gian delivery (build + push image)
- Tối ưu thời gian startup time (khởi động container) và shutdown time (tắt container)

### Basic Dockerfile
```
FROM node:14-alpine

# Đặt đường dẫn trong container nơi ta sẽ đưa code vào
WORKDIR /app

# Copy toàn bộ code ở thư mục hiện tại trên môi trường gốc -> vào đường dẫn hiện tại trong container (/app)
COPY . .

# Cài dependencies cho project -> build project -> start project
RUN yarn install
RUN yarn build
CMD ["yarn", "start"]
```

**Build without .dockerignore**
```
$ docker build -t example:basic -f .docker/basic.dockerfile .
```
<img width="967" alt="Screenshot 2024-01-29 at 15 15 32" src="https://github.com/khanhnguyendev/dockerize/assets/44081478/6b6538e0-7bf9-49fc-a18b-9c20a969b039">

**Build with .dockerignore**
> .dockerignore
```
node_modules
.next
.vscode
```
**Run Build**
```
$ docker build -t example:basic -f .docker/basic.dockerfile .
```
<img width="946" alt="Screenshot 2024-01-29 at 15 19 34" src="https://github.com/khanhnguyendev/dockerize/assets/44081478/adcfe423-6a7a-4802-ae01-9256b209f24b">

Đầu tiên các bạn tiến hành chỉnh sửa source
Sau đó ta tiến hành build lại image:
```
$ docker build -t example:basic -f .docker/basic.dockerfile .
```

## Analysis
- Project ví dụ ở đây là NextJS - cũng là project javascript như bao project khác, thành phần hay thay đổi nhất đó là source code, còn dependencies sẽ rất ít khi thay đổi
- Nhìn lại `basic.dockerfile`, bước copy toàn bộ source từ bên ngoài vào trong container lại được đặt ngay trên đầu.
  > Dẫn tới việc khi có bất kì thay đổi nào trong source code thì toàn bộ các bước từ đó trở về sau phải chạy lại

## Tận dụng docker layer caching
Từ những quan sát đó ta tổ chức lại Dockerfile 1 chút như sau:
  - Ban đầu chỉ cần copy file package.json và yarn.lock vào trong container để chạy yarn install là đủ để ta có node_modules
  - Sau đó copy source code vào.

### Basic Dockerfile with layer caching
```
FROM node:20-alpine

WORKDIR /app
COPY package.json ./

RUN yarn install
COPY . .
RUN yarn build
CMD ["yarn", "start"]
```

Tương tự như trên, ta tiến hành chỉnh sửa source sau đó build lại

```
$ docker build -t example:basic-cache -f .docker/basic-cached.dockerfile .
```
<img width="966" alt="Screenshot 2024-01-29 at 15 36 52" src="https://github.com/khanhnguyendev/dockerize/assets/44081478/cf730646-9966-4189-b467-a9cc799205a1">

## Compare result
> layer caching vs non-caching
<img width="1921" alt="Screenshot 2024-01-29 at 15 35 05" src="https://github.com/khanhnguyendev/dockerize/assets/44081478/865d8cda-e049-4f35-82a4-e2aecad17557">

> [!NOTE]
> `layer caching` build time nhanh hơn gấp 5 lần


## Tối giản size của Docker image
Đầu tiên ta kiểm tra size của image hiện tại
```
$ docker images | grep example
```

<img width="951" alt="Screenshot 2024-01-29 at 15 45 00" src="https://github.com/khanhnguyendev/dockerize/assets/44081478/e4cc4e5b-7da8-43d5-a3b1-972798fc20fd">

**Image size 2.15gb khá lớn :D**

> [!NOTE]
> khi push lên registry thường nó sẽ được nén lại size thực tế trên registry cỡ bằng 1/2->1/3



