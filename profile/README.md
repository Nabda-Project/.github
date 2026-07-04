<div align="center">

<img src="assets/brand/app-logo.png" width="140" alt="NABDA logo"/>

# NABDA — نبضة
### Intelligent Diagnostic Support & Follow-Up System

*A cardiovascular care ecosystem: wearable biosensing, conversational AI intake, and real-time doctor–patient follow-up.*

[![Org](https://img.shields.io/badge/organization-Nabda--Project-red?style=flat-square)](https://github.com/Nabda-Project)
[![Faculty](https://img.shields.io/badge/Alexandria%20University-Faculty%20of%20Engineering-003865?style=flat-square)](https://eng.alexu.edu.eg/)
[![Dept](https://img.shields.io/badge/Dept-Electronics%20%26%20Communications-informational?style=flat-square)](#)
[![Year](https://img.shields.io/badge/Academic%20Year-2025%2F2026-yellow?style=flat-square)](#)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](#license)

</div>

---

## About

**NABDA** ("pulse" in Arabic) is a graduation project from the Electronics & Communications program at Alexandria University's Faculty of Engineering. It targets a concrete problem in Egyptian healthcare: cardiac patients routinely wait 3+ hours for a consult, clinical communication is fragmented, and there is no continuous monitoring between visits.

NABDA closes that gap with four integrated pieces:

1. **An AI conversational agent** that intakes patients in dialectal Arabic before their appointment, so the doctor opens a ready-made case summary instead of starting from zero.
2. **A custom wearable device** (MAX30102 + ADS1292R on an ESP8266EX) that continuously tracks heart rate, SpO₂, and estimated blood pressure.
3. **A Spring Boot backend** that ingests vitals, evaluates them against clinical thresholds, stores the longitudinal patient record, and pushes real-time alerts.
4. **Two frontends** — a Flutter app for patients and doctors, and a Next.js web portal for doctors at a workstation — both talking to the same backend over REST and WebSocket/STOMP.

This repository (`.github`) is the organization's profile — it doesn't contain code, it explains how the six repositories below fit together.

---

## Table of Contents

- [System Architecture](#system-architecture)
- [How Data Flows](#how-data-flows)
  - [Pre-Consultation Chatbot Flow](#pre-consultation-chatbot-flow)
  - [Wearable Vitals & Alert Flow](#wearable-vitals--alert-flow)
- [Repositories](#repositories)
- [Repository Relationships](#repository-relationships)
- [Tech Stack](#tech-stack)
- [The Wearable Device](#the-wearable-device)
- [Team](#team)
- [Supervisor & Institution](#supervisor--institution)
- [Medical Disclaimer](#medical-disclaimer)
- [License](#license)

---

## System Architecture

Six subsystems communicate across REST, WebSocket/STOMP, UDP, and async job-polling channels:

```mermaid
flowchart LR
    subgraph IoT["IoT Layer"]
        WD["Nabda Wearable Device<br/>ESP8266EX + MAX30102 + ADS1292R"]
    end

    subgraph Clients["Client Layer"]
        MOB["Flutter Mobile App<br/>Patient + Doctor Portal"]
        WEB["Next.js Doctor Web Portal"]
        BOT["Chatbot Service<br/>Arabic intake (Flask)"]
    end

    subgraph Backend["Backend Layer"]
        BE["Spring Boot Backend<br/>REST + WebSocket/STOMP, JWT"]
    end

    subgraph Data["Data Layer"]
        DB[("PostgreSQL<br/>Amazon RDS, Multi-AZ")]
    end

    subgraph Cloud["Cloud Services"]
        AI["Report-model-api<br/>Fine-tuned Qwen3 (Azure, France Central)"]
        FCM["Firebase Cloud Messaging"]
    end

    WD -- "UDP, local Wi-Fi" --> MOB
    MOB -- "REST + WS/STOMP, JWT" --> BE
    WEB -- "REST + WS/STOMP, JWT" --> BE
    BOT -- "case summary sync" --> BE
    BOT -- "narrative / async poll" --> AI
    BE -- "JPA / SQL" --> DB
    BE -- "async REST relay" --> AI
    BE -- "FCM SDK" --> FCM
    FCM -. "push alerts" .-> MOB
    FCM -. "push alerts" .-> WEB
```

**Wearable Device** — samples vitals locally, runs sensor fusion on-device, and streams a UDP packet to the paired phone. It never talks to the backend directly and holds no persistent storage.

**Flutter Mobile App** — the primary patient interface and a full doctor portal for mobile use: auth, UDP listener, vitals dashboard, chatbot UI, AI report history, real-time chat, push notifications.

**Next.js Doctor Web Portal** — the same doctor feature set, built for a workstation. It talks to the identical REST/WebSocket API as the mobile app and never touches the database directly.

**Spring Boot Backend** — the orchestrator: stateless JWT auth, vitals ingestion and threshold evaluation, multi-channel alert dispatch (in-app + FCM + WebSocket), appointment and medical-record lifecycle, and the bridge to the external AI service.

**PostgreSQL (Amazon RDS)** — the single source of truth, accessed exclusively through Spring Data JPA/Hibernate.

**Report-model-api & Firebase** — external services: a fine-tuned Qwen3-4B model that turns an Arabic narrative into a structured clinical report, and FCM for cross-platform push delivery.

---

## How Data Flows

### Pre-Consultation Chatbot Flow

```mermaid
sequenceDiagram
    participant P as Patient
    participant Bot as Chatbot (Flask FSM)
    participant BE as Spring Boot Backend
    participant AI as Report-model-api (Qwen3)
    participant Doc as Doctor Dashboard

    P->>Bot: Answer structured intake questions (Arabic)
    Bot->>Bot: Build Arabic clinical narrative
    Bot->>AI: Submit narrative (async job)
    AI-->>Bot: job_id
    loop Poll until complete
        Bot->>AI: GET job status
    end
    AI-->>Bot: Structured clinical report
    Bot->>BE: Sync case summary + report
    BE->>BE: Persist AiConsultation
    BE-->>Doc: Report available before appointment
```

### Wearable Vitals & Alert Flow

```mermaid
sequenceDiagram
    participant WD as Wearable (ESP8266EX)
    participant App as Flutter App
    participant BE as Spring Boot Backend
    participant DB as PostgreSQL
    participant Doc as Doctor (Web/Mobile)

    WD->>App: UDP packet (fused HR, SpO2, BP)
    App->>App: Validate & normalize reading
    App->>BE: POST /api/iot/upload/{patientId}
    BE->>BE: Evaluate thresholds — NORMAL / WARNING / CRITICAL
    BE->>DB: Persist HealthMetric
    BE-->>App: WebSocket broadcast (live vitals)
    alt Reading is CRITICAL
        BE->>DB: Create Notification
        BE->>Doc: FCM push + WebSocket critical alert
    end
    BE-->>App: MeasurementResponse (status, priority)
```

---

## Repositories

| Repository | Role | Core Tech |
|---|---|---|
| [**gp_app**](https://github.com/Nabda-Project/gp_app) | Cross-platform mobile app for patients & doctors | Flutter, Dart, Firebase, Hive |
| [**gp-web-app**](https://github.com/Nabda-Project/gp-web-app) | Doctor-only web portal | Next.js 15, React 19, TypeScript, Tailwind |
| [**Graduation-project-BE**](https://github.com/Nabda-Project/Graduation-project-BE) | Central backend: auth, vitals, alerts, chat, records | Spring Boot 3.5, Java 21, PostgreSQL, JWT |
| [**Chatbot**](https://github.com/Nabda-Project/Chatbot) | Arabic conversational pre-consultation intake | Python, Flask |
| [**Report-model-api**](https://github.com/Nabda-Project/Report-model-api) | Fine-tuned LLM clinical report generation service | FastAPI, llama.cpp, Qwen3 (GGUF), Docker |
| [**Nabda-Device**](https://github.com/Gp-team26/Nabda-Device) | Wearable firmware, sensor fusion & PCB design | ESP8266 (Arduino), Altium Designer |

---

## Repository Relationships

```mermaid
graph TD
    ORG(("Nabda-Project"))
    ORG --> APP["gp_app<br/>Flutter mobile"]
    ORG --> WEB["gp-web-app<br/>Next.js web"]
    ORG --> BE["Graduation-project-BE<br/>Spring Boot backend"]
    ORG --> BOT["Chatbot<br/>Arabic intake (Flask)"]
    ORG --> AIREPO["Report-model-api<br/>Qwen3 report service"]
    ORG --> DEV["Nabda-Device<br/>ESP8266 firmware + PCB"]

    APP -->|"REST + WebSocket, JWT"| BE
    WEB -->|"REST + WebSocket, JWT"| BE
    DEV -->|"UDP, local Wi-Fi"| APP
    BOT -->|"case summary sync"| BE
    BOT -->|"narrative / async poll"| AIREPO
    BE -->|"async REST relay"| AIREPO
```

---

## Tech Stack

| Layer | Technologies |
|---|---|
| **Mobile** | Flutter, Dart, Hive, Flutter Secure Storage, Foreground Service (UDP listener) |
| **Web** | Next.js 15, React 19, TypeScript 5.8, Tailwind CSS, STOMP over SockJS |
| **Backend** | Spring Boot 3.5, Java 21, Spring Security (JWT), WebSocket/STOMP, MapStruct |
| **Database** | PostgreSQL 16 on Amazon RDS (Multi-AZ) |
| **AI / NLP** | Rule-based FSM chatbot, fine-tuned Qwen3-4B (GGUF via llama.cpp), JSON-schema-constrained generation |
| **Hardware** | ESP8266EX, MAX30102 (PPG), ADS1292R (ECG), custom Kalman-filter sensor fusion, Altium-designed PCB |
| **Cloud & DevOps** | AWS Elastic Beanstalk / EC2, Azure (France Central) inference, Firebase Cloud Messaging, GitHub Actions CI/CD, Docker |

---

## The Wearable Device

The Nabda wristband fuses PPG and ECG-derived signals through a custom Kalman filter, with a 3D-printed enclosure housing the PCB, LiPo battery, and sensors.

<table align="center">
  <tr>
    <td align="center" width="33%">
      <img src="assets/device/casing.png" width="260" alt="Nabda wearable casing render"/><br/>
      <sub>Enclosure render</sub>
    </td>
    <td align="center" width="33%">
      <img src="assets/device/prototype_top.png" width="260" alt="Nabda prototype top view"/><br/>
      <sub>Prototype — top view</sub>
    </td>
    <td align="center" width="33%">
      <img src="assets/device/prototype_bottom.png" width="260" alt="Nabda prototype bottom view"/><br/>
      <sub>Prototype — bottom view</sub>
    </td>
  </tr>
  <tr>
    <td align="center">
      <img src="assets/device/pcb-3d.png" width="260" alt="PCB 3D render, front"/><br/>
      <sub>PCB 3D render — front</sub>
    </td>
    <td align="center">
      <img src="assets/device/pcb-3d-back.png" width="260" alt="PCB 3D render, back"/><br/>
      <sub>PCB 3D render — back</sub>
    </td>
    <td align="center">
      <img src="assets/device/schematic.png" width="260" alt="PPG schematic"/><br/>
      <sub>PPG schematic</sub>
    </td>
  </tr>
</table>

---

## Team

Graduation Project Team 2025/2026 — Faculty of Engineering, Electronics & Communications, Alexandria University. Six equal contributors — listed alphabetically by first name, no seniority or hierarchy implied by order or position.

> Each card below has a photo slot. Drop a real headshot into `assets/team/<file>.png` (same filename, any size — it'll be cropped square) to replace the placeholder initials.

<table align="center">
  <tr>
    <td align="center" width="16.6%">
      <img src="assets/team/malak.png" width="95" height="95" alt="Malak Essam Kamal"/><br/><br/>
      <b>Malak Essam Kamal</b><br/>
      <a href="https://www.linkedin.com/in/malak-essam04/">LinkedIn ↗</a>
    </td>
    <td align="center" width="16.6%">
      <img src="assets/team/abdulshafi.png" width="95" height="95" alt="Mohammad Abdul-Shafi Seddiq"/><br/><br/>
      <b>Mohammad Abdul-Shafi Seddiq</b><br/>
      <a href="https://www.linkedin.com/in/abdulshafi/">LinkedIn ↗</a>
    </td>
    <td align="center" width="16.6%">
      <img src="assets/team/shahd.png" width="95" height="95" alt="Shahd Tamer Khamis"/><br/><br/>
      <b>Shahd Tamer Khamis</b><br/>
      <a href="https://www.linkedin.com/in/shahd-tamer-1b5303244/">LinkedIn ↗</a>
    </td>
    <td align="center" width="16.6%">
      <img src="assets/team/yehia.png" width="95" height="95" alt="Yehia Said Gewily"/><br/><br/>
      <b>Yehia Said Gewily</b><br/>
      <a href="https://www.linkedin.com/in/yehia-gewily-7545231a6/">LinkedIn ↗</a>
    </td>
    <td align="center" width="16.6%">
      <img src="assets/team/ziad-elsayed.png" width="95" height="95" alt="Ziad Mohammad Elsayed"/><br/><br/>
      <b>Ziad Mohammad Elsayed</b><br/>
      <a href="https://www.linkedin.com/in/ziadmohamedelsayed/">LinkedIn ↗</a>
    </td>
    <td align="center" width="16.6%">
      <img src="assets/team/ziad-zaki.png" width="95" height="95" alt="Ziad Mostafa Zaki"/><br/><br/>
      <b>Ziad Mostafa Zaki</b><br/>
      <a href="https://www.linkedin.com/in/ziad-mostafa-zaki-354952244/">LinkedIn ↗</a>
    </td>
  </tr>
</table>

---

## Supervisor & Institution

**Supervisor:** Dr. Aida El-Shafie

**Institution:** Alexandria University — Faculty of Engineering — Electrical Engineering Department, Electronics & Communications Major — Academic Year 2025/2026

---

## Medical Disclaimer

NABDA is a healthcare **support** system and a graduation-project prototype. It does not replace doctors, provide a final diagnosis, or make treatment decisions. All medical decisions must be made by qualified healthcare professionals. Clinical validation, regulatory approval, and larger-scale testing would be required before any real-world clinical use.

## License

Individual repositories are released under the MIT License unless otherwise noted in that repository. See each repo'
