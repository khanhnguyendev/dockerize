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
### 1/ Image Variants
**node:[version]**
- Use Case: Defacto image suitable for various purposes.
- Usage: Can be used as a throwaway container (mount your source code and start the container to run your app) or as a base for building other images.
- Note: Some tags may include Debian suite code names (e.g., bookworm, bullseye, buster), indicating the Debian release the image is based on. Explicitly specifying a suite code name can be useful if additional packages need to be installed.

**node:[version]-alpine:**
- Use Case: Focus on minimizing image size.
- Base: Built on Alpine Linux, a lightweight distribution (~5MB).
- Advantages: Results in slimmer images, suitable when reducing image size is a primary concern.
- Caveats: Uses musl libc instead of glibc, which might cause issues depending on libc requirements. Additional tools (e.g., git or bash) are uncommon in Alpine-based images, encouraging users to add only necessary components in their own Dockerfiles.

**node:[version]-slim:**
- Use Case: Emphasizes minimalism, containing only essential packages needed to run Node.js.
- Recommendation: Suitable for environments where only the Node.js image will be deployed, and space constraints are a concern.
- Note: Does not include common packages found in the default tag. However, unless there are specific constraints, using the default image is recommended.
  
<img width="930" alt="Screenshot 2024-02-01 at 14 06 58" src="https://github.com/khanhnguyendev/dockerize/assets/44081478/21e11743-cbdd-423c-a7c0-cee4cc3d7a62">

**Image size 2.15gb khá lớn :D**

> [!NOTE]
> khi push lên registry thường nó sẽ được nén lại size thực tế trên registry cỡ bằng 1/2->1/3

## 2/ Optimize with Multistage
Multi-stage build giúp bạn tạo ra một image production mà không phải giữ lại các devDependencies chỉ cần trong quá trình phát triển.
- `package.json` ta thấy rằng có rất nhiều packages ở đó, nhưng thực tế, sau bước yarn build thì số package ta thực tế cần không nhiều như thế, nhiều package -> node_modules sẽ to, thậm chí rất to -> size image to
- `yarn build` thì cái ta thực tế cần chỉ là folder .next hay public và node_modules mà thôi, các folder khác như pages, lib...(.eslint, .prettier...) không cần nữa
- `node_modules` chỉ cần trước lúc yarn build, sau đó thì vì ko cần nhiều package nữa nên ta chỉ cần node_modules dạng tí hon thôi

## 3/ Optimize `package.json`
- Hiện tại tất cả mọi package trong package.json đang được đặt ở dependencies, ta tách ra cái nào cần cho lúc dev ở local thì đưa nó vào devDependencies
- Reason:
https://docs.npmjs.com/cli/v8/commands/npm-install
<img width="829" alt="Screenshot 2024-02-01 at 14 01 07" src="https://github.com/khanhnguyendev/dockerize/assets/44081478/5ee9e086-11d7-4623-a9f5-4a47d38bfa1a">

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

Bằng cách này, image production chỉ sẽ chứa những gì cần thiết để chạy ứng dụng mà không có các devDependencies không cần thiết. Điều này giúp giảm kích thước của image và tăng tính bảo mật và hiệu suất trong môi trường production.

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

**Run build**
```
docker build -t example:multistage -f .docker/multistage.dockerfile .
```

<img width="949" alt="Screenshot 2024-01-29 at 16 24 25" src="https://github.com/khanhnguyendev/dockerize/assets/44081478/655140d0-ea8a-4312-8b87-f521ab0ac252">

**Stage deps (Install dependencies only when needed)**
- Tạo một giai đoạn tạm thời (deps) chỉ để cài đặt các dependencies từ package.json.
- Cài đặt dependencies dựa trên lockfile và đảm bảo rằng các dependencies không bị thay đổi giữa các lần build (--frozen-lockfile)
**Stage builder (Rebuild the source code only when needed)**
- Sử dụng stage `builder` để tái tạo mã nguồn chỉ khi cần thiết (khi có sự thay đổi trong source code).
- Copy mã nguồn và dependencies từ stage `deps`.
- Cài đặt dependencies trong môi trường production (yarn install --production --ignore-scripts --prefer-offline).
  (Không cài các devDependencies => điều này giúp giảm dung lượng của ảnh và tăng hiệu suất bằng cách chỉ build lại khi có sự thay đổi.)
**Stage runner (Production image)**
- Sử dụng stage `runner` để build production cuối cùng.
- Tạo user và group nextjs:nodejs với UID và GID là 1001 (for security reasons)
- Copy các tệp cần thiết từ giai đoạn builder.
- Run container with user nextjs.
  
**Result**
> [!IMPORTANT]
> Image size giảm đáng kể từ 2.15GB => 475MB (reduce ≈76.88%) 


