```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontSize': '16px', 'nodePadding': '12' }}}%%
flowchart LR
  %% C4 Level 2: Container Architecture Diagram (GitHub Native)

  subgraph Producers ["Upstream Producers & Clients"]
    restClients["<b>Synchronous Microservices</b><br/>(OTPs, Card Orders)"]
    kafkaProducer[("<b>Kafka Event Stream</b><br/>(Billing, Dunning Events)")]
    adminUI["<b>Campaign Admin UI</b><br/>(React / TypeScript)"]
  end

  subgraph PlatformBoundary ["Unified Notification Platform"]

    subgraph IngressLayer ["1. Ingress & Trigger Layer"]
      restApi["<b>REST Ingress API</b><br/>(Quarkus / Kotlin)<br/>Low-latency sync endpoint"]
      kafkaConsumer["<b>Kafka Event Consumer</b><br/>(Quarkus / Kotlin)<br/>Asynchronous event listener"]
      campaignApi["<b>Campaign Service</b><br/>(Node / Fastify)<br/>Batching & cohort scheduling"]
    end

    subgraph ProcessingLayer ["2. Processing & Validation Engine"]
      processor["<b>Core Processing Engine</b><br/>- Idempotency & Dedup Check<br/>- Profile & Locale Resolution<br/>- Pre-Send Coverage Check"]
    end

    subgraph LocalizationLayer ["3. Localization & Template Engine"]
      templateEngine["<b>MJML/Handlebars Compiler</b><br/>Compiles HTML & brand layouts"]
      lokaliseSync["<b>Lokalise Sync Worker</b><br/>Async translation fetcher"]
    end

    subgraph QueueLayer ["4. Dual-Priority Queueing & Dispatch"]
      highPriorityQ[("<b>High-Priority Queue</b><br/>Kafka Topic / InMemory<br/>(Transactional - < 2s SLA)")]
      lowPriorityQ[("<b>Low-Priority Queue</b><br/>Redis + BullMQ<br/>(Bulk / Throttled Campaigns)")]
      dispatcher["<b>Vendor Dispatcher</b><br/>Mailjet Send v3.1 Adapter<br/>(Rate limiting & retries)"]
    end

    subgraph FeedbackLayer ["5. Status & Webhook Reconciliation"]
      webhookHandler["<b>Webhook Receiver</b><br/>Ingests Mailjet events"]
      statusPublisher["<b>Status Reconciler</b><br/>Updates DB & emits Kafka status"]
    end

    subgraph StorageLayer ["6. Storage Layer"]
      postgres[("<b>PostgreSQL DB</b><br/>Metadata, outbox, status ledger")]
      redis[("<b>Redis Datastore</b><br/>Idempotency keys, Lokalise cache, BullMQ")]
      s3Store[("<b>S3 / Azure Blob Storage</b><br/>Rendered HTML/PDF audit snapshots")]
    end

  end

  subgraph SharedServices ["External Vendors & Shared Ecosystem"]
    customerApi["<b>Customer / Identity API</b>"]
    lokaliseApi["<b>Lokalise API</b><br/>(6 req/s limit)"]
    mailjetApi["<b>Mailjet Send API & Webhooks</b>"]
    kafkaStatusStream[("<b>Kafka Status Stream</b>")]
  end

  %% Ingress Flows
  restClients -->|"1a. HTTP POST"| restApi
  kafkaProducer -->|"1b. Consume Event"| kafkaConsumer
  adminUI -->|"1c. Schedule Campaign"| campaignApi

  restApi & kafkaConsumer & campaignApi -->|"2. Process Request"| processor

  %% Processing & Storage Interactions
  processor <-->|"Idempotency & Cache"| redis
  processor -->|"Write Outbox / Ledger"| postgres
  processor -->|"Fetch Profile"| customerApi

  %% Localization Flows
  processor -->|"3. Render Template"| templateEngine
  templateEngine <-->|"Read Translation Cache"| redis
  lokaliseSync -->|"Async Sync (Bypasses Limit)"| lokaliseApi
  lokaliseSync -->|"Populate Cache"| redis

  %% Queueing & Dispatch Flows
  templateEngine -->|"4a. High-Priority"| highPriorityQ
  templateEngine -->|"4b. Low-Priority"| lowPriorityQ
  templateEngine -->|"Save Snapshot (30d)"| s3Store

  highPriorityQ & lowPriorityQ -->|"5. Pull Jobs"| dispatcher
  dispatcher -->|"6. Send Email Payload"| mailjetApi

  %% Webhook Feedback Loop
  mailjetApi -->|"7. Webhook POST"| webhookHandler
  webhookHandler -->|"8. Reconcile"| statusPublisher
  statusPublisher -->|"Update Status"| postgres
  statusPublisher -->|"Publish Event"| kafkaStatusStream

  %% Styling
  classDef producerStyle fill:#084298,stroke:#084298,color:#fff;
  classDef containerStyle fill:#1168bd,stroke:#0b4884,color:#fff;
  classDef queueStyle fill:#d97706,stroke:#b45309,color:#fff;
  classDef dbStyle fill:#15803d,stroke:#166534,color:#fff;
  classDef extStyle fill:#6c757d,stroke:#495057,color:#fff;

  class restClients,kafkaProducer,adminUI producerStyle;
  class restApi,kafkaConsumer,campaignApi,processor,templateEngine,lokaliseSync,dispatcher,webhookHandler,statusPublisher containerStyle;
  class highPriorityQ,lowPriorityQ queueStyle;
  class postgres,redis,s3Store dbStyle;
  class customerApi,lokaliseApi,mailjetApi,kafkaStatusStream extStyle;
  ```
