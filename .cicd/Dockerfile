# Need to create a Dockerfile for java based applications
# Dockerfile for Eureka

FROM openjdk:18.0.2.1-jdk
ARG JAR_SOURCE
ENV JAR_SOURCE ${JAR_SOURCE}
ARG JAR_DEST
ENV JAR_DEST ${JAR_DEST}
RUN mkdir -p /opt/user/
WORKDIR /opt/user/
COPY ["${JAR_SOURCE}", "/opt/user/user.jar"]
RUN chmod 777 /opt/user/
EXPOSE 5736
CMD ["java", "-jar", "/opt/user/user.jar"]
