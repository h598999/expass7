# Expass 7
## -p and -e
docker run -p 5432:5432 \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydb \
  -d --name my-postgres --rm postgres

## Changing expass 4 to use postgres
I downloaded the psql cli client to interact with my postgresql server.
I started the postgresql server in a docker instance using this command:

docker run -p 5432:5432 \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydb \
  -d --name my-postgres --rm postgres

I then ran this line to create a user for the jpa:
``` SQL
CREATE USER jpa_client WITH PASSWORD 'secret';
```

Then i replaced the h2 lines in my persistence xml files 
with these:


``` XML
    ...
    <property name="hibernate.dialect" value="org.hibernate.dialect.PostgreSQLDialect"/>
    <property name="hibernate.connection.driver_class" value="org.postgresql.Driver"/>
    <property name="hibernate.connection.url" value="jdbc:postgresql://127.0.0.1:5432/postgres"/>
    <property name="hibernate.connection.username" value="jpa_client"/>
    <property name="hibernate.connection.password" value="secret"/>
    ...
```

Then i added these lines for sql bootstrapping:
``` XML
    ...
    <property name="jakarta.persistence.schema-generation.scripts.action" value="drop-and-create"/>
    <property name="jakarta.persistence.schema-generation.scripts.create-target" value="schema.up.sql"/>
    <property name="jakarta.persistence.schema-generation.scripts.drop-target" value="schema.down.sql"/>
    ...
```
Then i started this docker instance from a docker-compose.yml file:

``` yml
...
version: '3.8'
services:
postgres:
image: postgres:13
environment:
POSTGRES_DB: postgres
POSTGRES_USER: jpa_client
POSTGRES_PASSWORD: secret
ports:
- "5432:5432"
...
```
I then did a ./gradlew run.
This created the correct .sql files i then moved them to a init-scripts folder 
and then i started a docker instance from this docker-compose.yml file:


``` yml
...
version: '3.8'
services:
  db:
    image: postgres:13
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: jpa_client
      POSTGRES_PASSWORD: secret
    volumes:
      - ./init-scripts:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"

...
```
Then the tests worked fine

## Building your own docker application

I made this Dockerfile 

``` Dockerfile
FROM gradle:8.0.2-jdk17 AS build
WORKDIR /app
COPY . .
RUN gradle bootJar

FROM eclipse-temurin:17-jdk-alpine

# Set working directory
WORKDIR /app

COPY --from=build /app/build/libs/*.jar app.jar

# Expose the application port
EXPOSE 8080

# Run the application
CMD ["java", "-jar", "app.jar"]
```

I then ran this command to build the Docker image:
```bash
sudo docker build -t poll-app . 
```
That built a docker image named poll-app wihch is visible by running the:
```bash
sudo docker images
```
I then ran the command 
```bash
sudo docker run -p 8080:8080 poll-app
```
Which started the docker image and routed the localhost:8080 on the local computer
to the 8080 port in the docker instance
