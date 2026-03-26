# MOSIP Mock Services — Comprehensive Guide

[![Maven Package upon a push](https://github.com/mosip/mosip-mock-services/actions/workflows/push-trigger.yml/badge.svg?branch=master)](https://github.com/mosip/mosip-mock-services/actions/workflows/push-trigger.yml)
![License: MPL 2.0](https://img.shields.io/badge/License-MPL_2.0-brightgreen.svg)

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Repository Structure](#2-repository-structure)
3. [Architecture Overview](#3-architecture-overview)
4. [Global Prerequisites](#4-global-prerequisites)
5. [Module: MockMDS](#5-module-mockmds)
   - [Overview](#51-overview)
   - [Features](#52-features)
   - [Configuration](#53-configuration)
   - [Build & Run](#54-build--run)
   - [Admin API Reference](#55-admin-api-reference)
   - [Error Codes](#56-error-codes)
6. [Module: mock-abis](#6-module-mock-abis)
   - [Overview](#61-overview)
   - [Features](#62-features)
   - [Configuration](#63-configuration)
   - [Build & Run](#64-build--run)
   - [API Reference](#65-api-reference)
   - [Expectation Examples](#66-expectation-examples)
7. [Module: mock-sdk](#7-module-mock-sdk)
   - [Overview](#71-overview)
   - [Features](#72-features)
   - [Build & Integration](#73-build--integration)
8. [Module: mock-mv](#8-module-mock-mv)
   - [Overview](#81-overview)
   - [Features](#82-features)
   - [Configuration](#83-configuration)
   - [Build & Run](#84-build--run)
   - [API Reference](#85-api-reference)
9. [Module: softhsm](#9-module-softhsm)
   - [Overview](#91-overview)
   - [Build & Run](#92-build--run)
10. [Kubernetes & Helm Deployment](#10-kubernetes--helm-deployment)
11. [Contribution Guidelines](#11-contribution-guidelines)
12. [License](#12-license)

---

## 1. Introduction

**MOSIP Mock Services** is a collection of lightweight, configurable mock implementations of key components in the [MOSIP (Modular Open-Source Identity Platform)](https://mosip.io) ecosystem. These services are designed exclusively for **development, testing, and integration** environments and must **never** be used in production.

| Component | Purpose |
|-----------|---------|
| MockMDS | Simulates a biometric capture device (SBI-compliant) |
| mock-abis | Simulates an Automated Biometric Identification System |
| mock-sdk | Provides a mock biometric SDK library |
| mock-mv | Simulates a Manual Verification / Manual Adjudication service |
| softhsm | Simulates a Hardware Security Module using SoftHSM |

All Java-based modules use **Java 21**, **Spring Boot**, and **Maven 3.9.x**. They are built as self-contained JARs and (where applicable) shipped as Docker images.

---

## 2. Repository Structure

```
mosip-mock-services/
├── MockMDS/            # MOSIP Device Service mock (SBI spec)
├── mock-abis/          # Automated Biometric Identification System mock
├── mock-sdk/           # Biometric SDK mock library
├── mock-mv/            # Manual Verification / Adjudication mock
├── softhsm/            # SoftHSM-based Hardware Security Module mock
├── deploy/             # Shell scripts for Helm/K8s deployment
├── helm/               # Helm charts for mock-abis and mock-mv
├── Servers/            # (Additional server configurations)
├── LICENSE
├── NOTICE
└── README.md
```

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                  MOSIP Platform                      │
│                                                      │
│  Registration  ◄──── MockMDS (port 4501-4600)        │
│  Client                SBI endpoints                 │
│                                                      │
│  Registration  ◄──┐                                  │
│  Processor        │  ActiveMQ  ◄──► mock-abis        │
│                   │  Queues    ◄──► mock-mv           │
│                   └──────────────────────────────    │
│                                                      │
│  biosdk-services ◄──── mock-sdk (JAR dependency)     │
│                                                      │
│  Crypto Services ◄──── softhsm (PKCS#11, port 5666) │
└─────────────────────────────────────────────────────┘
```

- **MockMDS** communicates directly with MOSIP's Registration Client over HTTP on configurable local ports.
- **mock-abis** and **mock-mv** communicate asynchronously via **ActiveMQ** message queues; both also expose REST APIs over HTTP.
- **mock-sdk** is a JAR library consumed by [biosdk-services](https://github.com/mosip/biosdk-services).
- **softhsm** exposes a PKCS#11 interface over TCP (port 5666) for cryptographic operations.

---

## 4. Global Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Java (JDK) | 21 | Required by all Java modules |
| Maven | 3.9.x | Build tool |
| Git | Any recent | To clone the repository |
| Docker | Any recent | Required for containerized deployment |
| Docker Compose | v2+ | For running multi-container setups |
| Postman | Optional | For manual API testing |

**Clone the repository:**
```bash
git clone https://github.com/mosip/mosip-mock-services.git
cd mosip-mock-services
```

---

## 5. Module: MockMDS

### 5.1 Overview

MockMDS is a mock implementation of the **MOSIP Device Service (MDS)**, conforming to the [Secure Biometric Interface (SBI) specification](https://docs.mosip.io/1.2.0/modules/partner-management-services/pms-existing/device-provider-partner#sbi-secure-biometric-interface). It serves biometric capture data from local files and is typically run on a developer's workstation (Windows or Linux/Mac) to simulate actual biometric hardware.

It runs as a Spring Boot application listening on a configurable port range (default: **4501–4600**).

### 5.2 Features

| Feature | Description |
|---------|-------------|
| SBI Compliance | Implements Device Discovery (`MOSIPDISC`), Device Info (`MOSIPDINFO`), Capture, RCapture, and Stream endpoints |
| Dual Mode | Supports **Registration** and **Authentication** device profiles |
| Configurable Status | Set device to `Ready`, `Busy`, `Not Ready`, or `Not Registered` |
| Configurable Quality | Set biometric quality score (0–100) |
| Response Delay | Inject artificial delays for specific API methods |
| Profile Switching | Switch between named biometric data profiles (`Default`, `Profile1`, `Profile2`, …) |
| Multi-Modality | Supports Face, Finger (Slap/Single), and Iris (Double/Single) modalities |
| File-Based Simulation | Reads biometric data from local files |

### 5.3 Configuration

All configuration lives in `MockMDS/application.properties`.

#### Server Settings

```properties
server.minport=4501
server.maxport=4600
server.serveripaddress=127.0.0.1

cors.headers.allowed.methods="OPTIONS, RCAPTURE, CAPTURE, MOSIPDINFO, MOSIPDISC, STREAM, GET, POST"
cors.headers.allowed.origin="*"
```

#### Device File Paths (example for Face modality)

```properties
mosip.mock.sbi.file.face.digitalid.json=/Biometric Devices/Face/DigitalId.json
mosip.mock.sbi.file.face.deviceinfo.json=/Biometric Devices/Face/DeviceInfo.json
mosip.mock.sbi.file.face.devicediscovery.json=/Biometric Devices/Face/DeviceDiscovery.json
mosip.mock.sbi.file.face.streamimage=/Biometric Devices/Face/Stream Image/0.jpeg
mosip.mock.sbi.file.face.keys.keystorefilename=/Biometric Devices/Face/Keys/mosipface.p12
mosip.mock.sbi.file.face.keys.keyalias=mosipface
mosip.mock.sbi.file.face.keys.keystorepwd=mosipface
# FTM key
mosip.mock.sbi.file.face.keys.keystorefilename.ftm=/Biometric Devices/Face/Keys/mosipfaceftm.p12
mosip.mock.sbi.file.face.keys.keyalias.ftm=mosipfaceftm
mosip.mock.sbi.file.face.keys.keystorepwd.ftm=mosipfaceftm
```

Similar properties exist for `finger.slap`, `finger.single`, `iris.double`, and `iris.single` modalities.

#### MOSIP Server Integration

```properties
mosip.auth.server.url=https://{env}/v1/authmanager/authenticate/clientidsecretkey
mosip.auth.appid=regproc
mosip.auth.clientid=mosip-regproc-client
mosip.auth.secretkey={password}

mosip.ida.server.url=https://{env}/idauthentication/v1/internal/getCertificate?applicationId=IDA&referenceId=IDA-FIR
```

#### Placing Certificates

Place the device-partner and FTM keystore files (`.p12`) in the appropriate modality keys directory:

```
Biometric Devices/{Modality}/Keys/
  ├── device-partner.p12
  └── ftm-partner.p12
```

### 5.4 Build & Run

#### Build

```bash
cd MockMDS
mvn clean install -Dmaven.test.skip=true -Dgpg.skip=true
```

#### Run — Authentication Mode

Using script (Windows):
```bat
target\run_auth.bat
```

Using Java directly:
```bash
java -cp "mock-mds-1.3.0-SNAPSHOT.jar;lib\*" io.mosip.mock.sbi.test.TestMockSBI \
  "mosip.mock.sbi.device.purpose=Auth" \
  "mosip.mock.sbi.biometric.type=Biometric Device" \
  "mosip.mock.sbi.biometric.image.type=WSQ"
```

#### Run — Registration Mode

Using script (Windows):
```bat
target\run_reg.bat
```

Using Java directly:
```bash
java -cp "mock-mds-1.3.0-SNAPSHOT.jar;lib\*" io.mosip.mock.sbi.test.TestMockSBI \
  "mosip.mock.sbi.device.purpose=Registration" \
  "mosip.mock.sbi.biometric.type=Biometric Device"
```

> **Note:** MockMDS does not ship a Dockerfile because MDS typically runs on the client machine (Windows or Android), not on a server.

### 5.5 Admin API Reference

The service exposes admin endpoints to control mock behavior at runtime:

| Endpoint | Method | Description | Request Body |
|----------|--------|-------------|-------------|
| `/admin/status` | POST | Set device status | `{"type": "Biometric Device", "deviceStatus": "Ready"}` |
| `/admin/score` | POST | Set quality score | `{"type": "Biometric Device", "qualityScore": "44.44", "fromIso": false}` |
| `/admin/delay` | POST | Set response delay (ms) | `{"type": "Biometric Device", "delay": "10000", "method": ["RCAPTURE"]}` |
| `/admin/profile` | POST | Set biometric data profile | `{"type": "Biometric Device", "profileId": "Profile1"}` |

**Valid values:**

| Field | Valid Values |
|-------|-------------|
| `deviceStatus` | `Ready`, `Busy`, `Not Ready`, `Not Registered` |
| `method` (delay) | `MOSIPDISC`, `MOSIPDINFO`, `CAPTURE`, `STREAM`, `RCAPTURE` |
| `profileId` | `Default`, `Profile1`, `Profile2` (and any custom profiles placed in `/Profile/`) |

### 5.6 Error Codes

| Code | Message |
|------|---------|
| 0 | Success |
| 100 | Device not registered |
| 101 | Unable to detect a biometric object |
| 102 | Technical error during extraction |
| 103 | Device tamper detected |
| 104 | Unable to connect to management server |
| 105 | Image orientation error |
| 106 | Device not found |
| 107 | Device public key expired |
| 108 | Domain public key missing |
| 109 | Requested number of biometrics not supported |
| 110 | Device is not ready |
| 111 | Device is busy |
| 112 | Invalid Transaction ID |
| 113 | Count mismatch for given deviceType |
| 114 | Device Type mismatch (`Finger`, `Iris`, `Face`) |
| 115 | Env values mismatch (`Staging`, `Developer`, `Pre-Production`, `Production`) |
| 116 | Count value mismatch |
| 117 | Exception value mismatch |
| 118 | Exception mismatch with type of biometric data |
| 119 | Bio SubType value mismatch |
| 120 | Device Type mismatch for given deviceId |
| 121 | Purpose value mismatch (`Registration`, `Auth`) |
| 122 | BioSubType/Exception mismatch for given biometric type |
| 500 | Invalid URL |
| 501 | Invalid Type value in Device Discovery Request |
| 502 | Biometric Type values mismatch |
| 503 | Devices are not connected |
| 504 | Device Status values mismatch |
| 505 | Quality Score validation error |
| 506 | Delay cannot be negative |
| 507 | Invalid method in delay request |
| 551 | Profile not set |
| 601–610 | Livestream errors |
| 700–710 | RCapture errors |
| 800–810 | Auth Capture errors |
| 999 | Unknown error |

---

## 6. Module: mock-abis

### 6.1 Overview

`mock-abis` is a mock implementation of the **Automated Biometric Identification System (ABIS)**. It integrates with the MOSIP Registration Processor via ActiveMQ message queues, simulating biometric **Insert** and **Identify** (deduplication) operations. It also exposes REST APIs and a Swagger UI for configuring expectations and triggering test scenarios.

- **Artifact**: `mock-abis-1.3.0-SNAPSHOT.jar`
- **Default port**: `8081`
- **Swagger UI**: `http://localhost:8081/v1/mock-abis-service/swagger-ui/index.html#/`

### 6.2 Features

| Feature | Description |
|---------|-------------|
| ABIS Simulation | Processes `Insert` and `Identify` operations via ActiveMQ |
| Configurable Expectations | Per-biometric-hash: force Success/Error/Duplicate, set error codes, inject delays |
| Partner Encryption | Supports CBEFF encryption using partner certificates (`cbeff.p12`) |
| ActiveMQ Integration | Consumes from `inboundQueueName`, publishes to `outboundQueueName` |
| Swagger Interface | Interactive UI to manage expectations and configuration |
| H2 In-Memory DB | No external DB required for local development |
| Prometheus Metrics | Micrometer integration via `/actuator/prometheus` |

### 6.3 Configuration

#### ActiveMQ / Queue Configuration

Create `src/main/resources/registration-processor-abis.json` from the provided sample template:

```json
{
  "abis": [
    {
      "name": "ABIS",
      "host": "",
      "port": "",
      "brokerUrl": "tcp://{env}.mosip.net:{port}",
      "inboundQueueName": "ctk-to-abis",
      "outboundQueueName": "abis-to-ctk",
      "pingInboundQueueName": "ctk-to-abis",
      "pingOutboundQueueName": "abis-to-ctk",
      "userName": "artemis",
      "password": "{password}",
      "typeOfQueue": "ACTIVEMQ",
      "inboundMessageTTL": 2700
    }
  ]
}
```

For a **fully local** setup, run the bundled ActiveMQ container:

```bash
cd mock-abis/activemq
docker-compose up
# Access UI at: http://localhost:8161/
```

#### H2 Database (Local Development)

No additional setup is required. Configuration in `application.properties`:

```properties
javax.persistence.jdbc.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
javax.persistence.jdbc.driver=org.h2.Driver
javax.persistence.jdbc.user=sa
javax.persistence.jdbc.password=sa
hibernate.ddl-auto=update
hibernate.dialect=org.hibernate.dialect.H2Dialect
```

#### Partner Certificate (Optional)

Place `cbeff.p12` in `src/main/resources/` to enable partner-based encryption. The certificate can also be uploaded at runtime via the Swagger upload endpoint.

### 6.4 Build & Run

#### Build

```bash
cd mock-abis
mvn clean install -Dgpg.skip=true
```

#### Run with Docker (Recommended)

Using the provided Docker Compose file:

```bash
docker-compose -f sample-docker-compose.yml up
```

`sample-docker-compose.yml`:

```yaml
version: '3'
services:
  mock_abis:
    build: .
    image: mock-abis
    container_name: mock-abis
    ports:
      - "8081:8081"
    environment:
      - active_profile_env=local
      - spring_config_label_env=develop
      - spring_config_url_env=localhost
    volumes:
      - "~/keystore:/home/${container_user}/keystore"
    restart: always
```

#### Build & Run Docker Image Manually

```bash
# Build
docker build --file Dockerfile --tag mock-abis .

# Run locally
docker run -p 8081:8081 mock-abis

# Push to registry
docker push <your-registry>/mock-abis:latest
```

#### Run as JAR (Local Development)

1. Download `kernel-auth-adapter-1.3.0-SNAPSHOT.jar` and place in a `lib/` folder.
2. Create and configure `registration-processor-abis.json` (see above).
3. Build: `mvn clean install -Dgpg.skip=true`
4. Run:

```bash
java -Dloader.path=lib/kernel-auth-adapter-1.3.0-SNAPSHOT.jar \
  -Dlocal.development=true \
  -Dabis.bio.encryption=true \
  -Dspring.profiles.active=local \
  -Dmosip_host=https://<server-hostname> \
  --add-opens java.xml/jdk.xml.internal=ALL-UNNAMED \
  --add-opens java.base/java.lang.reflect=ALL-UNNAMED \
  --add-opens java.base/java.lang.stream=ALL-UNNAMED \
  --add-opens java.base/java.time=ALL-UNNAMED \
  -jar target/mock-abis-1.3.0-SNAPSHOT.jar
```

**JVM Flags Reference:**

| Flag | Description |
|------|-------------|
| `local.development=true` | Loads `registration-processor-abis.json` from classpath resources |
| `abis.bio.encryption=true` | Enables partner-based CBEFF encryption (requires `cbeff.p12`) |
| `mosip_host` | Hostname of the target MOSIP server |
| `spring.profiles.active=local` | Activates local profile |

### 6.5 API Reference

All endpoints are prefixed with `http://{host}/v1/mock-abis-service/`.

#### Configuration APIs

##### GET Configuration
```
GET /config/configure
```
Response:
```json
{ "findDuplicate": false }
```

##### POST — Update Configuration
```
POST /config/configure
Content-Type: application/json

{ "findDuplicate": "false" }
```
Response: `Successfully updated the configuration`

#### Expectation APIs

##### POST — Set Expectation
```
POST /config/expectation
Content-Type: application/json
```
Request body:
```json
{
  "id": "<SHA256 hash of ISO biometric>",
  "version": "[xxxxx]",
  "requesttime": "2021-05-05T05:44:58.525Z",
  "actionToInterfere": "Identify",
  "forcedResponse": "Error",
  "errorCode": "4",
  "delayInExecution": "5",
  "gallery": {
    "referenceIds": [
      { "referenceId": "<hash of duplicate biometric>" }
    ]
  }
}
```

| Field | Values |
|-------|--------|
| `actionToInterfere` | `Insert`, `Identify` |
| `forcedResponse` | `Error`, `Success`, `Duplicate` |
| `errorCode` | Any numeric ABIS error code string |
| `delayInExecution` | Seconds as string (e.g., `"5"`) |

##### GET — List All Expectations
```
GET /config/expectation
```

##### DELETE — Remove Expectation
```
DELETE /config/expectation/{id}
```
Response: `Successfully deleted expectation $expectation_id`

### 6.6 Expectation Examples

#### Insert — Return Error with Delay

```json
{
  "id": "b57038791f59de0d43cd9c06dd2c621888d3170ed725ca9cb7f70122519b8484",
  "version": "xxxxx",
  "requesttime": "2021-05-05T05:44:58.525Z",
  "actionToInterfere": "Insert",
  "forcedResponse": "Error",
  "errorCode": "4",
  "delayInExecution": "5",
  "gallery": null
}
```

#### Identify — Return Duplicate

```json
{
  "id": "b57038791f59de0d43cd9c06dd2c621888d3170ed725ca9cb7f70122519b8484",
  "version": "xxxxx",
  "requesttime": "2021-05-05T05:44:58.525Z",
  "actionToInterfere": "Identify",
  "forcedResponse": "Duplicate",
  "errorCode": "",
  "delayInExecution": "",
  "gallery": {
    "referenceIds": [
      { "referenceId": "bbe2a7932e9f228c21504a63a5ef1b829ad99478d0ca3b75a159b7342260314a" }
    ]
  }
}
```

> **Tip:** The `id` field is the SHA-256 hash of the decoded ISO image:
> ```
> SHA256(base64_decode(bdb))
> ```
> Use the "get cached biometrics" API to verify hashes.

---

## 7. Module: mock-sdk

### 7.1 Overview

`mock-sdk` is a **library JAR** (not a standalone service) that provides a mock implementation of the [`IBioAPIV2`](https://github.com/mosip/bio-utils/blob/master/kernel-biometrics-api/src/main/java/io/mosip/kernel/biometrics/spi/IBioApiV2.java) interface. It is designed to be used as a dependency by [`biosdk-services`](https://github.com/mosip/biosdk-services), enabling biometric operations without real SDK hardware.

- **Artifact**: `mock-sdk-1.3.0-SNAPSHOT.jar`
- **Main class**: `io.mosip.mock.sdk.impl.SampleSDKV2`
- **No standalone deployment** — consumed as a Maven dependency or JAR reference.

### 7.2 Features

| Feature | Description |
|---------|-------------|
| 1:N Match | Mock biometric matching across a gallery |
| Segmentation | Simulates biometric image segmentation |
| Extraction | Simulates template extraction from biometric images |
| Quality Check | Returns configurable quality scores |
| Format Conversion | Simulates conversion between biometric formats (e.g., WSQ, ISO) |
| IBioAPIV2 Compliance | Fully implements the MOSIP biometric SDK interface |

**Implemented Services:**
- `MatchService` — 1:N matching logic
- `CheckQualityService` — Quality scoring
- `SegmentService` — Segmentation
- `ExtractTemplateService` — Template extraction
- `ConvertFormatService` — Format conversion
- `SDKInfoService` — SDK information

### 7.3 Build & Integration

#### Build Locally

```bash
cd mock-sdk
mvn clean install -Dgpg.skip=true
```

This installs `mock-sdk-1.3.0-SNAPSHOT.jar` to your local Maven repository.

#### Integration with biosdk-services

Add the following properties to your `biosdk-services` configuration:

```properties
biosdk_class=io.mosip.mock.sdk.impl.SampleSDKV2
mosip.role.biosdk.getservicestatus=REGISTRATION_PROCESSOR
biosdk_bioapi_impl=io.mosip.mock.sdk.impl.SampleSDKV2
```

#### Upgrade

```bash
git pull
# Update version in pom.xml if needed
mvn clean install -Dgpg.skip=true
```

---

## 8. Module: mock-mv

### 8.1 Overview

`mock-mv` is a mock implementation of the **Manual Verification (MV)** / **Manual Adjudication** service in MOSIP's Registration Processor pipeline. It listens to an ActiveMQ queue for verification requests and automatically publishes configurable APPROVED or REJECTED responses.

- **Artifact**: `mock-mv-1.3.0-SNAPSHOT.jar`
- **Default port**: configurable via Spring Cloud Config
- **Swagger URL**: `https://<hostname>/v1/mockmv/swagger-ui.html#/`

### 8.2 Features

| Feature | Description |
|---------|-------------|
| MV Simulation | Automatically responds APPROVED or REJECTED to adjudication requests |
| Queue Integration | Consumes from and publishes to ActiveMQ queues |
| Per-RID Expectations | Override default decision for specific registration IDs (RIDs) |
| Response Delay | Simulate delayed manual review |
| Swagger Interface | Configure behavior via REST API |

### 8.3 Configuration

Configure ActiveMQ queues and default decisions via `application.properties`:

```properties
# Default adjudication decision
mock.mv.decision=APPROVED
```

The service reads queue connection details from Spring Cloud Config. For local development, ensure your config server provides the ActiveMQ `brokerUrl`, queue names, and credentials.

### 8.4 Build & Run

#### Build

```bash
cd mock-mv
mvn clean install -Dgpg.skip=true
```

#### Run with Docker (Sandbox / Server)

```bash
# Build image
docker build . --file Dockerfile --tag mock-mv

# Push to registry
docker push <your-registry>/mock-mv:latest
```

#### Run as JAR (Local Development)

```bash
java \
  -XX:-UseG1GC -XX:-UseParallelGC -XX:-UseShenandoahGC \
  -XX:+ExplicitGCInvokesConcurrent -XX:+UseZGC -XX:+ZGenerational \
  -XX:+UnlockExperimentalVMOptions -XX:+UseStringDeduplication \
  -XX:+HeapDumpOnOutOfMemoryError -XX:+UseCompressedOops \
  -XX:MaxGCPauseMillis=200 \
  -Dfile.encoding=UTF-8 \
  -Dspring.cloud.config.label="master" \
  -Dspring.profiles.active="default" \
  -Dspring.cloud.config.uri="http://localhost:51000/config" \
  --add-opens java.xml/jdk.xml.internal=ALL-UNNAMED \
  --add-opens java.base/java.lang.reflect=ALL-UNNAMED \
  --add-opens java.base/java.lang.stream=ALL-UNNAMED \
  --add-opens java.base/java.time=ALL-UNNAMED \
  -jar target/mock-mv-1.3.0-SNAPSHOT.jar
```

> Adjust `spring.cloud.config.uri` to point to your local or remote config server.

### 8.5 API Reference

All endpoints are prefixed with `http://{host}/v1/mockmv/`.

#### Configuration APIs

##### GET Configuration
```
GET /config/configureMockMv
```
Response:
```json
{ "mockMvDescision": "APPROVED" }
```

##### POST — Update Configuration
```
POST /config/configureMockMv
Content-Type: application/json

{ "mockMvDescision": "APPROVED" }
```
Response: `Successfully updated the configuration`

#### Expectation APIs

##### POST — Set Per-RID Expectation
```
POST /config/expectationMockMv
Content-Type: application/json
```
Request body:
```json
{
  "mockMvDecision": "REJECTED",
  "delayResponse": 30,
  "rid": "10332103161016320241119230824"
}
```

| Field | Values | Notes |
|-------|--------|-------|
| `mockMvDecision` | `APPROVED`, `REJECTED` | Decision for this RID |
| `delayResponse` | Integer (seconds) | Delay before sending response |
| `rid` | String | Registration ID to match |

Response: `Successfully inserted expectation $expectation_id`

##### GET — List All Expectations
```
GET /config/expectationMockMv
```
Response:
```json
{
  "10332103161016320241119230824": {
    "mockMvDecision": "REJECTED",
    "delayResponse": 0,
    "rid": "10332103161016320241119230824"
  }
}
```

##### DELETE — Remove Expectation
```
DELETE /config/expectation/{id}
```
Response: `Successfully deleted expectation $expectation_id`

---

## 9. Module: softhsm

### 9.1 Overview

`softhsm` provides a **network-accessible Hardware Security Module (HSM)** simulator using [SoftHSM](https://www.opendnssec.org/softhsm/). It exposes a PKCS#11 interface over TCP on port **5666**, enabling cryptographic operations without physical HSM hardware.

The module also includes:
- An **artifactory** container (nginx) to serve the PKCS#11 client library (`client.zip`)
- A **test client** container demonstrating how to connect to the HSM over the network

### 9.2 Build & Run

#### Build

```bash
cd softhsm
./dev-build.sh 1111
```

Replace `1111` with your desired security PIN.

#### Step 1 — Start the SoftHSM Server

```bash
docker run -p 5666:5666 softhsm:v1
```

Optionally, copy the compiled PKCS#11 client library:

```bash
docker cp <container_id>:/client.zip .
```

Only build the artifactory container after extracting the new `client.zip`.

#### Step 2 — Start the Artifactory (Optional — for testing only)

```bash
docker run -p 80:80 artifactory:v1
```

> The `client.zip` is served from the hardcoded path: `artifactory/libs-release-local/hsm/client.zip`

#### Step 3 — Start the HSM Client

```bash
docker run \
  -e ARTIFACTORY_URL="http://<artifactory-host>" \
  -e PKCS11_PROXY_SOCKET="tcp://<softhsm-host>:5666" \
  hsmclient:v1
```

Replace `<softhsm-host>` with the IP address of the machine running the SoftHSM container.

#### Publishing

Once testing is complete, push the `softhsm` image to the MOSIP Docker registry and publish `client.zip` to the actual Artifactory instance.

---

## 10. Kubernetes & Helm Deployment

Helm charts for `mock-abis` and `mock-mv` are available in the `helm/` directory. The `deploy/` directory contains helper scripts for installation and management.

### Available Scripts

| Script | Description |
|--------|-------------|
| `deploy/install.sh` | Installs mock services (mock-abis, mock-mv, mock-smtp, mock-sms) via Helm |
| `deploy/delete.sh` | Removes deployed Helm releases |
| `deploy/restart.sh` | Restarts running deployments |
| `deploy/copy_cm.sh` | Copies ConfigMaps between namespaces |

### Deploy to Kubernetes

```bash
cd deploy
./install.sh
```

This deploys:
- mock-abis
- mock-mv
- Mock SMTP
- Mock SMS

> For full sandbox deployment instructions, refer to [MOSIP V3 Installation Guide](https://docs.mosip.io/1.2.0/deploymentnew/v3-installation).

---

## 11. Contribution Guidelines

We welcome contributions from the community!

- **Code Contributions**: See [MOSIP Code Contribution Guide](https://docs.mosip.io/1.2.0/community/code-contributions)
- **Community Forum**: Post questions and issues on the [MOSIP Community Forum](https://community.mosip.io/)
- **GitHub Issues**: [![GitHub Issues](https://img.shields.io/badge/GitHub-Issues-181717?style=flat&logo=github&logoColor=white)](https://github.com/mosip/mosip-mock-services/issues)

### Development Tips

- Use `-Dspring.profiles.active=local` to activate the local profile for mock-abis and mock-mv.
- For mock-abis, pass `mosip_host=https://<mosip-host>` as an environment variable when connecting to a remote server's queue.
- Always use `registration-processor-abis.json` (copied from `registration-processor-abis-sample.json`) for local queue config.
- For biometric hash computation: `SHA256(base64_decode(bdb))` — never hash the raw BDB directly.

---

## 12. License

![License: MPL 2.0](https://img.shields.io/badge/License-MPL_2.0-brightgreen.svg)

This project is licensed under the [Mozilla Public License 2.0 (MPL 2.0)](LICENSE).

> **Important:** These mock services are intended exclusively for **development and testing** environments. Do **not** deploy them in production — replace each mock with the corresponding real component before going live.
