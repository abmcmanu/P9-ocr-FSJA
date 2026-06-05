# ── Stage 1 : build ──────────────────────────────────────────────────────────
FROM gradle:8.7-jdk17 AS build

WORKDIR /src
# Copie uniquement les fichiers de configuration Gradle en premier
# pour profiter du cache Docker lors des builds successifs
COPY build.gradle settings.gradle gradlew ./
COPY gradle/ gradle/
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon || true

COPY src/ src/
RUN ./gradlew build -x test --no-daemon

# ── Stage 2 : image d'exécution ──────────────────────────────────────────────
FROM amazoncorretto:17-alpine AS runtime

# Utilisateur non-root pour la sécurité
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY --from=build /src/build/libs/microcrm-0.0.1-SNAPSHOT.jar app.jar

RUN chown appuser:appgroup app.jar

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
