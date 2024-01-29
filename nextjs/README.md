# Dockerfile: Optimize build time and image size

## Tại sao phải Omtimize Dockerfile
- Tối ưu chi phí lưu trữ (container registry)
- Tối ưu thời gian delivery (build + push image)
- Tối ưu thời gian startup time (khởi động container) và shutdown time (tắt container)

### Basic Dockerfile
```
FROM node:20-alpine

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

## Analysis
- `package.json` ta thấy rằng có rất nhiều packages ở đó, nhưng thực tế, sau bước yarn build thì số package ta thực tế cần không nhiều như thế, nhiều package -> node_modules sẽ to, thậm chí rất to -> size image to
- `yarn build` thì cái ta thực tế cần chỉ là folder .next hay public và node_modules mà thôi, các folder khác như pages, lib...(.eslint, .prettier...) không cần nữa
- `node_modules` chỉ cần trước lúc yarn build, sau đó thì vì ko cần nhiều package nữa nên ta chỉ cần node_modules dạng tí hon thôi

## Optimize with Multistage
- `package.json`
Hiện tại tất cả mọi package trong package.json đang được đặt ở dependencies, ta tách ra cái nào cần cho lúc dev ở local thì đưa nó vào devDependencies, lát nữa yarn build xong thì loại bỏ nó khỏi node_modules
- `Multistages`
Các stage ta chỉ COPY những thứ thật cần thiết của stage trước đó làm "gốc" cho stage hiện tại

> Current package.json
```
{
  "name": "nextjs-example",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "react": "^18",
    "react-dom": "^18",
    "next": "14.1.0",
    "typescript": "^5",
    "@types/node": "^20",
    "@types/react": "^18",
    "@types/react-dom": "^18",
    "autoprefixer": "^10.0.1",
    "postcss": "^8",
    "tailwindcss": "^3.3.0",
    "eslint": "^8",
    "eslint-config-next": "14.1.0"
  },
  "devDependencies": {
  }
}
```
> Optimized package.json
```
{
  "name": "nextjs-example",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "react": "^18",
    "react-dom": "^18",
    "next": "14.1.0"
  },
  "devDependencies": {
    "typescript": "^5",
    "@types/node": "^20",
    "@types/react": "^18",
    "@types/react-dom": "^18",
    "autoprefixer": "^10.0.1",
    "postcss": "^8",
    "tailwindcss": "^3.3.0",
    "eslint": "^8",
    "eslint-config-next": "14.1.0"
  }
}
```

### Multistage Dockerfile
```
# Install dependencies only when needed
FROM node:20-alpine AS deps

WORKDIR /app
COPY package.json ./
RUN yarn install --frozen-lockfile

# Rebuild the source code only when needed
FROM node:20-alpine AS builder
WORKDIR /app
COPY . .
COPY --from=deps /app/node_modules ./node_modules
RUN yarn build && yarn install --production --ignore-scripts --prefer-offline

# Production image, copy all the files and run next
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV production

RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json

USER nextjs

CMD ["yarn", "start"]
```

**Explanation**
Ta có tất cả 3 stages:
- `deps`: chỉ chạy yarn install mục đích là để ta có được folder node_modules
- `builder`: ở đây ta sẽ lấy folder node_modules từ stage deps và tiến hành build project, ngay sau khi build ta cũng chạy lại yarn install 1 lần nữa với option --production ý bảo yarn là chỉ giữ lại những package nào được khai báo ở dependencies còn cái nào thuộc devDependencies thì loại hết nó ra khỏi node_modules (bước này giảm size đi đáng kể đó 😉)
- `runner`: COPY lấy các thành phần thật sự cần thiết cho production từ stage builder và chạy project lên.
  - Ở đây ta cũng tạo user nextjs với UID:GID=1001:1001 để chạy project (luôn dùng user non-root để chạy app production - for security reasons)

**Run build**
```
docker build -t example:multistage -f .docker/multistage.dockerfile .
```

<img width="949" alt="Screenshot 2024-01-29 at 16 24 25" src="https://github.com/khanhnguyendev/dockerize/assets/44081478/655140d0-ea8a-4312-8b87-f521ab0ac252">


**Result**
> [!IMPORTANT]
> Image size giảm đáng kể từ 2.15GB => 475MB (reduce ≈76.88%) 


