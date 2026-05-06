# 1. build stage
FROM node:alpine3.22 AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build


# 2. production stage (nginx)
FROM nginx:stable-alpine3.21

# SPA 설정 (Vue Router 대응)
COPY default.conf /etc/nginx/conf.d/default.conf

# 빌드 결과물
COPY --from=builder /app/dist /usr/share/nginx/html

ENV TZ=Asia/Seoul

EXPOSE 85

CMD ["nginx", "-g", "daemon off;"]