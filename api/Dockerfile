FROM eclipse-temurin:17-jdk-alpine
FROM eclipse-temurin:17-jre-alpine
# Copy resource from previous stage
COPY --from=0 Main.class .

# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY simpleapi/pom.xml .
COPY simpleapi/src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar

