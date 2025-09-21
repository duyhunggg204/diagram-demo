flowchart LR
  %% ===== Edge / Sources =====
  subgraph EDGE["Clinic LAN / Edge"]
    A1["Workstations & Kiosk (Win/Linux)"]
    A2["EHR/HIS App Servers"]
    A3["Databases (Audit ON)"]
    A4["IdP/SSO Agent, EDR/MDM"]
    F["Fluent Bit Agents"]
    A1 -->|Windows Event / Syslog| F
    A2 -->|App Audit / API| F
    A3 -->|DB Audit (SELECT / UPDATE)| F
  end

  %% ===== Ingest =====
  F --|mTLS / TLS1.2+|--> G["NGINX Ingest Gateway<br/>(mTLS, rate-limit, allowlist)"]
  G --> V["Vector (Normalize/Mask PII/PHI)<br/>(optional)"]

  %% ===== Buffer/Stream =====
  V --> Q["Redis Streams or NATS JetStream"]
  G -.-> Q

  %% ===== Analyzer =====
  Q --> P["Analyzer API (FastAPI)"]
  P --> R["Rules Engine (Sigma / OPA Rego)"]
  P --> AD["Anomaly Scoring (z-score / IsolationForest)"]
  P --> E[(PostgreSQL<br/>Events / Findings / Configs / Rule Versions)]
  P --> L["Loki Log Store<br/>(raw / near-raw, labels)"]
  P --> AL["Alerting: Email / Slack / Teams"]

  %% ===== Observability & Reports =====
  subgraph OBS["Observability & Reporting"]
    L --> GR["Grafana Dashboards"]
    E --> GR
    E --> CE["Celery Scheduler<br/>(Reports PDF, Audit Pack)"]
  end

  %% ===== Security Foundations =====
  subgraph SEC["Security Foundations"]
    KMS["Vault / Cloud KMS<br/>(Certs/Keys, PKI, HSM)"]
    NTP["NTP (time sync)"]
  end
  KMS --- G
  KMS --- P
  NTP --- F
  NTP --- L
  NTP --- E

  %% ===== Optional Security Controls =====
  subgraph OPT["Optional Security Controls"]
    WZ["Wazuh / OSQuery (FIM, EDR-like)"]
    DLP["DLP / Egress Proxy (policy, object lock)"]
  end
  A1 --- WZ
  A2 --- WZ
  G --- DLP
