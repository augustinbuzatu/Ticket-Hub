# Ticket-Hub
TicketHub is a Java &amp; Spring Boot ticketing app (REST) that uses constraints to handle multiple requests at once.

## Run locally

Requirements: Java 25 and Docker.

1. Create the local configuration from the example (the default values work as they are):

   ```bash
   cp .env.example .env
   ```

2. Start PostgreSQL:

   ```bash
   docker compose up -d
   ```

3. Start the application:

   ```bash
   ./mvnw spring-boot:run
   ```

The application listens on port 8080:

- Health check: http://localhost:8080/actuator/health
- Swagger UI: http://localhost:8080/swagger-ui.html

`docker compose down` stops the database and keeps its data in a Docker volume;
`docker compose down -v` deletes the data as well.
