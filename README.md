# Java Microservice Setup

This project is a complete Java-based microservices architecture using **Spring Boot 3.2.0** and **Spring Cloud 2023.0.0** on **Java 21**. It showcases Service Discovery, Routing, Load Balancing, and dynamic route redirection using Spring Cloud components.

---

## Architecture Overview

The system consists of four independent microservices that interact with each other:

```
                  ┌────────────────────────┐
                  │     Eureka Server      │ (Service Registry)
                  │      (Port: 8761)      │
                  └───────────▲────────────┘
                              │
            Registers &       │ Registers &
            Fetches Registry  │ Fetches Registry
                              │
┌───────────┐             ┌───┴────────────┐             ┌───────────┐
│ Client /  │────────────►│  API Gateway   │────────────►│ Service A │ (Port: 8081)
│ Browser   │             │  (Port: 8080)  │             │   [WAR]   │
└───────────┘             └───┬────────────┘             └───────────┘
                              │
                              │ Dynamic Route
                              ▼
                         ┌───────────┐
                         │ Service B │ (Port: 8082)
                         │   [WAR]   │
                         └───────────┘
```

- **Service Registry (Eureka)**: Acts as the central registry where all services register themselves and lookup other services.
- **API Gateway**: Serves as the single entry point for all client requests, routing traffic dynamically to the correct services using logical service IDs registered in Eureka.
- **Service A & Service B**: Core business logic microservices packaged as Web Archives (WARs).

An visual overview of the setup is available in the root folder: [image.png](file:///d:/Antigravity/Projects/java-microservice-setup/image.png).

---

## Component Details

### 1. Eureka Server (`eurekaServer`)
* **Port**: `8761`
* **Packaging**: `JAR`
* **Main Annotation**: `@EnableEurekaServer`
* **Key Configuration (`application.properties`)**:
  ```properties
  server.port=8761
  spring.application.name=eureka-server
  
  # Disable self-registration and registry fetching as this is the server
  eureka.client.register-with-eureka=false
  eureka.client.fetch-registry=false
  ```

---

### 2. API Gateway (`gateway`)
* **Port**: `8080`
* **Packaging**: `JAR`
* **Main Annotation**: `@EnableDiscoveryClient`
* **Key Configuration (`application.properties`)**:
  * **Registry Connection**: Connects to the Eureka Server at `http://localhost:8761/eureka/`.
  * **Routing Rules**:
    * Requests matching `/service-a/**` are dynamically load-balanced and routed to `SERVICE-A` (`lb://SERVICE-A`).
    * Requests matching `/service-b/**` are dynamically load-balanced and routed to `SERVICE-B` (`lb://SERVICE-B`).
  * **Network Resilience**: Uses custom `inetutils` configurations to ignore virtual networks (e.g., Docker, WSL `vEthernet`) and prioritize local IP subnets (`192.168`, `10`).

---

### 3. Service A (`service-a`)
* **Port**: `8081`
* **Packaging**: `WAR`
* **Main Annotation**: `@EnableDiscoveryClient`
* **Controller Path**: `/service-a`
* **Exposed Endpoints**:
  * `GET /service-a/hello` -> Returns `"Hello from Service A!"`
* **Key Configuration (`application.properties`)**:
  * Connects to Eureka for registration (`http://localhost:8761/eureka/`).
  * Self-identifies as `service-a`.
  * Includes `inetutils` network binding rules to ensure the correct host IP is registered with Eureka.

---

### 4. Service B (`service-b`)
* **Port**: `8082`
* **Packaging**: `WAR`
* **Main Annotation**: `@EnableDiscoveryClient`
* **Controller Path**: `/service-b`
* **Exposed Endpoints**:
  * `GET /service-b/hello` -> Returns `"Hello from Service B!"`
* **Key Configuration (`application.properties`)**:
  * Connects to Eureka for registration (`http://localhost:8761/eureka/`).
  * Self-identifies as `service-b`.
  * Configured with network interface filters matching Service A.

---

## Build & Package Instructions

Each project uses **Apache Maven** and can be packaged individually.

1. Navigate to any of the service directories:
   * [eurekaServer](file:///d:/Antigravity/Projects/java-microservice-setup/eurekaServer)
   * [gateway](file:///d:/Antigravity/Projects/java-microservice-setup/gateway)
   * [service-a](file:///d:/Antigravity/Projects/java-microservice-setup/service-a)
   * [service-b](file:///d:/Antigravity/Projects/java-microservice-setup/service-b)

2. Build and package the binaries:
   ```bash
   mvn clean package
   ```
   * *Note:* Service A and Service B will output `.war` files in their respective `target` directories, while Eureka Server and the Gateway will output `.jar` files.

---

## Running the Architecture

To start the system, services should be brought up in the following order:

1. **Start Eureka Server**:
   ```bash
   cd eurekaServer
   mvn spring-boot:run
   ```
2. **Start Service A & Service B**:
   ```bash
   cd ../service-a
   mvn spring-boot:run
   ```
   ```bash
   cd ../service-b
   mvn spring-boot:run
   ```
3. **Start API Gateway**:
   ```bash
   cd ../gateway
   mvn spring-boot:run
   ```

---

## Verifying the Setup

### 1. Eureka Dashboard
Once everything is started, navigate to the Eureka dashboard:
* **URL**: [http://localhost:8761](http://localhost:8761)
* Under **"Instances currently registered with Eureka"**, you should see:
  * `API-GATEWAY` (on Port 8080)
  * `SERVICE-A` (on Port 8081)
  * `SERVICE-B` (on Port 8082)

### 2. Testing API Gateway Routing
Clients do not need to make requests directly to the individual service ports (`8081` / `8082`). Instead, they route everything through the API Gateway (Port `8080`):

* **Route to Service A**:
  * **Endpoint**: [http://localhost:8080/service-a/hello](http://localhost:8080/service-a/hello)
  * **Expected Response**: `Hello from Service A!`
  
* **Route to Service B**:
  * **Endpoint**: [http://localhost:8080/service-b/hello](http://localhost:8080/service-b/hello)
  * **Expected Response**: `Hello from Service B!`
