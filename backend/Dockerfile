# 1) build stage
FROM maven:3.9.9-eclipse-temurin-21-alpine AS builder

WORKDIR /build

COPY . .

RUN mvn clean package -DskipTests


# 2) runtime stage
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY --from=builder /build/target/*.jar app.jar

ARG BUILD_PROFILE=local
ENV APP_PROFILE=${BUILD_PROFILE}
ENV TZ=Asia/Seoul

EXPOSE 8085

ENTRYPOINT ["sh", "-c", "java -jar app.jar --spring.profiles.active=${APP_PROFILE}"]