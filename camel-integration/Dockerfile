# =============================================================================
# Multi-stage Dockerfile for Camel Quarkus Integration
# =============================================================================
# Supports both JVM and native builds
# Usage:
#   JVM build:    docker build -t camel-integration:latest .
#   Native build: docker build -t camel-integration:latest --build-arg BUILD_TYPE=native .
# =============================================================================

ARG BUILD_TYPE=jvm

# =============================================================================
# Stage 1: Build stage
# =============================================================================
FROM registry.access.redhat.com/ubi8/openjdk-21:1.20 AS builder

ARG BUILD_TYPE

USER root
WORKDIR /build

# Install Maven
RUN microdnf install -y maven && microdnf clean all

# Copy pom.xml first for dependency caching
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source code
COPY src ./src

# Build based on type
RUN if [ "$BUILD_TYPE" = "native" ]; then \
        mvn package -Pnative -DskipTests -Dquarkus.native.container-build=true; \
    else \
        mvn package -DskipTests; \
    fi

# =============================================================================
# Stage 2a: JVM Runtime
# =============================================================================
FROM registry.access.redhat.com/ubi8/openjdk-21-runtime:1.20 AS jvm-runtime

ARG BUILD_TYPE

ENV LANGUAGE='en_US:en'
ENV JAVA_OPTS_APPEND="-Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager"
ENV JAVA_APP_JAR="/deployments/quarkus-run.jar"

# Configure the JAVA_APP_JAR
COPY --from=builder --chown=185 /build/target/quarkus-app/lib/ /deployments/lib/
COPY --from=builder --chown=185 /build/target/quarkus-app/*.jar /deployments/
COPY --from=builder --chown=185 /build/target/quarkus-app/app/ /deployments/app/
COPY --from=builder --chown=185 /build/target/quarkus-app/quarkus/ /deployments/quarkus/

EXPOSE 8080 9000
USER 185

ENTRYPOINT [ "/opt/jboss/container/java/run/run-java.sh" ]

# =============================================================================
# Stage 2b: Native Runtime
# =============================================================================
FROM registry.access.redhat.com/ubi8/ubi-minimal:8.10 AS native-runtime

WORKDIR /work/
RUN chown 1001 /work \
    && chmod "g+rwX" /work \
    && chown 1001:root /work

COPY --from=builder --chown=1001:root /build/target/*-runner /work/application

EXPOSE 8080 9000
USER 1001

ENTRYPOINT ["./application", "-Dquarkus.http.host=0.0.0.0"]

# =============================================================================
# Final stage selection
# =============================================================================
FROM ${BUILD_TYPE}-runtime AS final

LABEL io.k8s.display-name="File Pipeline Camel Integration" \
      io.k8s.description="Camel Quarkus integration for file transfer and document processing" \
      io.openshift.tags="camel,quarkus,integration" \
      maintainer="File Pipeline Team"

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:9000/q/health/live || exit 1
