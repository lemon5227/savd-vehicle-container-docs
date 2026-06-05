# Eclipse Leda: Seat Adjuster Example Application

This document provides a comprehensive overview of the `seat-adjuster` vehicle application, detailing its architecture, underlying technologies, and the explicit deployment sequence required to run it successfully inside an integrated Software-Defined Vehicle (SDV) environment.

source : https://github.com/eclipse-leda/leda-example-applications/tree/main/seat-adjuster

---

## 1. Functional Overview

The **Seat Adjuster** application is a prototypical Software-Defined Vehicle (SDV) service that demonstrates how cloud-to-device or off-board telemetry can safely interact with core physical vehicle functions. 

The application exposes an asynchronous interface to receive target seating modifications via an MQTT message broker, acts on those requests by evaluating constraints, and pushes state transitions downstream to a centralized data broker managing the standardized vehicle state.

### Key Workflows:
1. **State Observation**: On boot, the application subscribes to the localized Vehicle Signal Specification (VSS) path for driver seat placement (`Vehicle.Cabin.Seat.Row1.DriverSide.Position`).
2. **Command Processing**: The application listens on a specific MQTT control topic for ingress target positions (`seatadjuster/setPosition/request`).
3. **Actuation Simulation**: When a payload arrives, it verifies the coordinates, issues an actuation command, and updates the state within the vehicle's telemetry engine.

---

## 2. System Architecture & Components

The environment utilizes a containerized microservices approach deployed over an isolated Docker bridge network (`sdv-network`). It relies on three primary building blocks from the Eclipse SDV ecosystem:

| Component | Technology | Description |
| :--- | :--- | :--- |
| **Vehicle Application** | `seat-adjuster-app` | A Python-based microservice built using the **Eclipse Velocitas SDK**. Contains the core application logic, structural constraints, and state manipulation handling. |
| **Vehicle Abstraction** | `KUKSA Databroker` | A high-performance gRPC service implementing the **Vehicle Signal Specification (VSS)** data model. It functions as the single source of truth for all sensor, actuator, and attribute data streams across the vehicle. |
| **Messaging Layer** | `Eclipse Mosquitto` | An open-source **MQTT message broker** handling lightweight pub/sub messaging patterns. It bridges communication between external entities (e.g., mobile apps, test runners) and the internal application code. |

### Crucial Integration Note: Protocol Versioning
Modern versions of the KUKSA Databroker deprecate the legacy `sdv.databroker.v1` gRPC protocol interface by default. Because the embedded Velocitas Python SDK within this version of the seat-adjuster relies on these legacy entrypoints, the Databroker must be explicitly initialized to tolerate and serve V1 protocol requests via the `--enable-databroker-v1` flag.

---

## 3. Complete Deployment Guide

Run the following commands sequentially from the terminal inside your repository directory (`~/Desktop/sdv_project/leda-example-applications/seat-adjuster`) to spin up the entire stack from a clean state.

### Step 1: Environment Sanitation
Ensure no lingering zombie containers or named configurations from previous test runs conflict with network bindings or ports:

```bash
docker rm -f databroker mosquitto seat-adjuster 2>/dev/null
docker network create sdv-network 2>/dev/null || true

```

### Step 2: Build the Local Vehicle Application Image

Compile the local Docker context into a deployable container image named `seat-adjuster-app`:

```bash
docker build --load -t seat-adjuster-app .

```

### Step 3: Launch the KUKSA Databroker

Deploy the vehicle signal registry with insecure access allowed for testing, and the legacy V1 gRPC protocol subsystem explicitly enabled:

```bash
docker run -d --name databroker \
  --network sdv-network \
  -p 55555:55555 \
  ghcr.io/eclipse-kuksa/kuksa-databroker:latest \
  --insecure --enable-databroker-v1

```

### Step 4: Launch the Mosquitto MQTT Broker

Bind the messaging system to the common bridge network and map standard MQTT ports out to the host system while loading the necessary local configuration maps:

```bash
docker run -d --name mosquitto \
  --network sdv-network \
  -p 1883:1883 \
  -v $(pwd)/mosquitto-conf:/mosquitto/config \
  eclipse-mosquitto:latest

```

### Step 5: Launch the Seat Adjuster Application

Boot the core python runtime, supplying explicit environment variables mapping how it can discover and securely bind to the containerized infrastructure endpoints over the network mesh:

```bash
docker run -d --name seat-adjuster \
  --network sdv-network \
  -e SDV_MQTT_ADDRESS="mqtt://mosquitto:1883" \
  -e SDV_VEHICLEDATABROKER_ADDRESS="grpc://databroker:55555" \
  seat-adjuster-app

```

---

## 4. Operational Testing & Verification

Because the utilities `mosquitto_sub` and `mosquitto_pub` are natively packed inside the containerized broker image, testing can be performed instantly via `docker exec` without forcing dependencies onto your host operating system environment.

### 1. Initialize Event Monitoring (Terminal Window 1)

Open a dedicated shell window to mirror internal messaging telemetry. This sets a persistent hook tracking any commands routed under the `seatadjuster` top-level topic namespace:

```bash
docker exec -it mosquitto mosquitto_sub -t 'seatadjuster/#' -v

```

### 2. Dispatch Actuation Ingress (Terminal Window 2)

In a secondary shell, issue a mock control request payload specifying a new target position parameter (e.g., target setting `600` on a standard structural scale):

```bash
docker exec -it mosquitto mosquitto_pub \
  -t seatadjuster/setPosition/request \
  -m '{"position": 600, "requestId": "12345"}'

```

### Expected Output

Upon successful distribution of the payload, Terminal Window 1 will instantly populate with the matching JSON event payload, followed by the Velocitas application output acknowledging the validation check and executing the state transition into the KUKSA Databroker layer.

---

## 5. Troubleshooting Utilities

If data streams appear dropped or locked, use the following operational hooks to pull direct component logs:

* **Review Vehicle App Stack Traces:**
```bash
docker logs seat-adjuster

```


*Healthy state confirmation lookahead:* Verify that the trailing line outputs `Mqtt native connection OK!` with no active Python `AioRpcError` tracebacks.
* **Inspect Broker Connection Handshakes:**
```bash
docker logs mosquitto

```

## 6. Stoping properly : 

For a "break" : 
```bash
docker stop seat-adjuster mosquitto databroker
```
Followed by this to restart : 
```bash
docker start seat-adjuster mosquitto databroker
```

At the end of the testing
```bash
docker rm -f seat-adjuster mosquitto databroker
```