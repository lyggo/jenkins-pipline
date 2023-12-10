# Use the official OpenJDK 8 image as the base image
FROM openjdk:8-jdk-alpine

# Set the working directory in the container
WORKDIR /app

# Copy the JAR file into the container
COPY target/xgame-web.jar /app/xgame-web.jar

# Copy the JaCoCo agent JAR into the container
COPY jacocoagent.jar /app/jacocoagent.jar

# Expose the port that your Spring Boot application will run on
EXPOSE 8080

# Set default JVM options as environment variables
ENV DEFAULT_JAVA_OPTS="-Dspring.cloud.nacos.config.namespace=xxxxxx \
               -Dspring.cloud.nacos.config.server-addr=127.0.0.1:xxxx \
               -Dspring.cloud.nacos.discovery.namespace=xxxxxx \
               -Dspring.cloud.nacos.discovery.server-addr=127.0.0.1:xxxx \
               -Dspring.profiles.active=daily \
               -Dspring.application.name=xgame-web-100 \
               -Dserver.port=8080"

# Allow additional JAVA_OPTS and INCLUDES to be passed during container runtime
ENV JAVA_OPTS=${DEFAULT_JAVA_OPTS}
ENV INCLUDES="*"

# Create a directory to store the logs and coverage data
RUN mkdir -p /app/logs /app/reports

# Entry point command to run your application with JaCoCo agent and TCP server
CMD sh -c 'java $JAVA_OPTS -javaagent:jacocoagent.jar=destfile=/app/reports/jacoco.exec,append=true,includes=$INCLUDES,tcpserver,port=6300 -jar xgame-web.jar 2>&1 | tee /app/logs/xgame-web.log'