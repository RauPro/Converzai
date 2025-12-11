# Converza: Architectural Design Package

## 1. Project Overview

**Converza** is a Call Performance Intelligence Platform that turns the "black box" of customer conversations into clear, actionable insights.

The platform is built for businesses that rely on phone-based sales and support. The system connects with Twilio to ingest live calls, uses AI to analyze revenue outcomes (like upsells and conversions) and sentiment, and delivers those metrics in a unified dashboard for management.

This project serves as a proof-of-concept for the "Insight Engine"—the core pipeline that transforms raw audio into structured business intelligence.

---

## 2. System Elements

### 2.1 Users
*   **Manager (Primary):** The main user. They log in to track team performance, review specific call recordings, and analyze ROI metrics.
*   **System Admin (Internal):** Responsible for tenant management and monitoring API usage (OpenAI/Twilio) to keep the system healthy and cost-effective.
*   **Agent (Passive):** The employees making the calls. They don't log in directly but exist as data points linked to performance metrics.

### 2.2 Frontends
*   **Manager Dashboard (Web App):** A Next.js (React) application that serves as the main interface for the "Performance Insight Dashboard".
*   **Onboarding Portal:** A public-facing flow where managers can easily provision their Twilio numbers to kick off the pilot.

### 2.3 Backends
*   **Core Logic Service (Python/FastAPI):** The brain of the operation. It handles Twilio webhooks, cleans up transcripts, and manages prompt engineering for the OpenAI integration.
*   **BaaS / Data Service (Supabase):** The system uses Supabase to handle Authentication, the Database (PostgreSQL), and Object Storage, avoiding the need to reinvent the wheel on infrastructure.

### 2.4 External Integrations
*   **Twilio (Telephony & STT):** Handles the actual call recording and initial Speech-to-Text transcription.
*   **OpenAI (Intelligence Engine):** The GPT-4o API is used to analyze unstructured text for sentiment, conversion success, and script adherence.

### 2.5 Local Data Store & Entities
**Database:** PostgreSQL (via Supabase).

**Key Entities:**
*   `Tenants` (Organizations)
*   `Users` (Managers)
*   `Agents` (Employees)
*   `Calls` (Metadata like duration and timestamps)
*   `Transcripts` (Raw text)
*   `Insights` (AI outputs: Sentiment, Conversion Boolean, Upsell $)

---

## 3. Architectural Patterns

### 3.1 High-Level Pattern: Event-Driven Service Oriented
The system adopts a hybrid **Service-Oriented Architecture (SOA)**. It is not a monolith, but avoids overcomplication with full microservices.

*   **Frontend-Backend Separation:** The Next.js frontend is completely decoupled from the logic and communicates via REST.
*   **Event-Driven Pipeline:** The core value (analysis) is triggered by external events (webhooks) rather than direct user requests.

### 3.2 Decomposition Pattern
*   **Backend as a Service (BaaS):** Supabase is leveraged for "commoditized" features like Auth and standard CRUD APIs to speed up MVP delivery.
*   **Specialized Compute Service:** The Python FastAPI service acts as a specialized worker node for the "Call Analysis Pipeline." This is separated because it requires heavy compute libraries and has different scaling needs compared to simple CRUD operations.

### 3.3 Scaling Strategy
*   **Horizontal Scaling:** The Python Logic Service is stateless. As call volume grows, more FastAPI instances can be added behind a load balancer to handle the influx of Twilio webhooks.
*   **Asynchronous Processing:** By decoupling call completion (Webhook) from analysis (AI Processing), the system ensures it doesn't lock up during traffic spikes.

---

## 4. Intra-system Communication

### 4.1 Communication Styles

| Interaction | Style | Protocol | Reasoning |
| :--- | :--- | :--- | :--- |
| **Dashboard $\leftrightarrow$ Supabase** | **Synchronous** | REST/HTTPS | Managers need to see their data immediately. |
| **Twilio $\rightarrow$ Logic Service** | **Asynchronous** | Webhook (HTTP) | Twilio pushes data when a call ends; the system acknowledges receipt instantly and processes it in the background. |
| **Logic Service $\rightarrow$ OpenAI** | **Synchronous** | REST/HTTPS | The service waits for the LLM response, but this happens inside a background worker so the main thread stays free. |
| **Logic Service $\rightarrow$ Database** | **Synchronous** | SQL/TCP | Analysis results are written directly to Postgres once processing is done. |

### 4.2 The Asynchronous Call Pipeline (Core Feature)
1.  **Trigger:** Twilio sends a `POST` webhook to indicate a recording is ready.
2.  **Ack:** FastAPI responds with a `200 OK` immediately (latency < 200ms).
3.  **Process:** A background task kicks off:
    *   Fetch the transcript from Twilio.
    *   Send it to OpenAI (GPT-4o).
    *   Parse the JSON response.
    *   Write `Call_Insights` to Supabase.
4.  **Update:** The Dashboard (Next.js) re-fetches the data or receives a real-time update via Supabase to show the new stats.

---

## 5. Authentication & Authorization (AuthN/AuthZ)

### 5.1 Authentication (AuthN)
*   **User Login:** Handled entirely by **Supabase Auth**. Managers get a JSON Web Token (JWT) when they log in.
*   **Service Verification:** The FastAPI Logic Service authenticates requests from Twilio by validating the **X-Twilio-Signature** header. This ensures only expensive AI pipelines are run for legitimate Twilio calls.

### 5.2 Authorization (AuthZ)
*   **Multi-Tenancy (Row Level Security):** This is critical. A manager from "Company A" should never see calls from "Company B." This is enforced using strict **PostgreSQL RLS policies** in Supabase.
    *   *Rule:* `SELECT * FROM calls WHERE tenant_id = auth.uid().organization_id`
*   **Role-Based Access:**
    *   **Admins:** Global Read/Write.
    *   **Managers:** Scoped Read/Write (Own Tenant only).
    *   **Public (Unauthenticated):** Access restricted to Marketing and Login pages.
## 6. Architecture

```mermaid
graph TD
    %% --- STYLING ---
    classDef person fill:#f9f,stroke:#333,stroke-width:2px,color:black;
    classDef frontend fill:#ffdec2,stroke:#d67e2c,stroke-width:2px,color:black;
    classDef backend fill:#c2e0ff,stroke:#2c5bb5,stroke-width:2px,color:black;
    classDef db fill:#c2e0ff,stroke:#2c5bb5,stroke-width:2px,shape:cylinder,color:black;
    classDef external fill:#d4edda,stroke:#28a745,stroke-width:2px,stroke-dasharray: 5 5,color:black;
    classDef legend fill:#fff,stroke:#333,stroke-width:1px,color:black;

    %% --- LEGEND ---
    subgraph Legend
        L_User((User)):::person
        L_Front("User-Facing Frontend"):::frontend
        L_Back("Internal Backend"):::backend
        L_Ext("External Service"):::external
        L_Line1[Solid: Synchronous HTTPS] --- L_Line2[Dotted: Asynchronous / Background]
    end

    %% --- USERS ---
    Manager((Manager)):::person
    Admin((System Admin)):::person
    Agent(("Agent / Caller")):::person

    %% --- EXTERNAL SYSTEMS (PUBLIC INTERNET) ---
    Twilio["Twilio<br/>(Telephony & STT)"]:::external
    OpenAI["OpenAI<br/>(GPT-4o API)"]:::external

    %% --- CONVERZA CLOUD INFRASTRUCTURE ---
    subgraph Converza_Cloud ["Converza Cloud Environment"]
        style Converza_Cloud fill:#f4f4f4,stroke:#666,stroke-dasharray: 5 5

        %% Frontend Layer (Publicly Accessible via URL)
        subgraph Public_Zone ["Public Availability Zone"]
            style Public_Zone fill:#fff,stroke:#d67e2c
            NextApp["<b>Dashboard & Onboarding</b><br/>Next.js / React"]:::frontend
        end

        %% Backend Layer (Internal / Protected)
        subgraph Private_Zone ["Private Availability Zone"]
            style Private_Zone fill:#fff,stroke:#2c5bb5
            
            FastAPI["<b>Core Logic Service</b><br/>Python FastAPI<br/><i>Async Worker</i>"]:::backend
            
            subgraph BaaS ["Supabase (BaaS)"]
                SupabaseAuth["<b>Auth Service</b><br/>JWT Handling"]:::backend
                SupabaseDB[("<b>PostgreSQL DB</b><br/>RLS Policies")]:::db
            end
        end
    end

    %% --- INTERACTIONS & DATA FLOW ---

    %% 1. Telephony Flow (The Trigger)
    Agent -.->|Phone Call Audio| Twilio
    Twilio --"Webhook (POST)<br/>[Auth: X-Twilio-Signature]"--> FastAPI

    %% 2. Manager Interaction
    Manager == "HTTPS / Browser<br/>[AuthN: Login]" ==> NextApp
    NextApp -- "Auth Request" --> SupabaseAuth
    SupabaseAuth -- "JWT Token" --> NextApp
    NextApp == "REST API (Fetch KPIs)<br/>[AuthZ: JWT + RLS]" ==> SupabaseDB

    %% 3. Admin Interaction
    Admin == "HTTPS / Browser<br/>[AuthZ: Global Access]" ==> NextApp

    %% 4. The Intelligence Pipeline (Internal/Async)
    FastAPI -- "Fetch Transcript (Sync)" --> Twilio
    FastAPI -- "Analyze Text (Sync)" --> OpenAI
    OpenAI -- "JSON Insights" --> FastAPI
    FastAPI -- "Write Insights (Sync)<br/>[AuthZ: Service Key]" --> SupabaseDB

    %% Link Legend to graph to keep it tidy (Hidden link)
    Legend ~~~ Manager
```
## 7. User Flows
* Manager Authentication: A manager logs in to access their secure environment.
* Frictionless Onboarding (Twilio Provisioning): A manager selects and provisions a phone number to begin the pilot immediately.
* Agent Call Handling (Passive): An agent accepts a call on the provisioned line; the system passively records it.
* Call Completion & Webhook Trigger: Twilio detects the call end and notifies our system to begin ingestion.
* AI Insight Generation: The system processes the raw transcript to extract ROI metrics (Upsell $, Sentiment) using GPT-4o.
* Dashboard KPI Retrieval: The manager loads the dashboard to view aggregated performance metrics (Conversion Rate, Revenue).
* Coaching Drill-Down: The manager selects a specific low-performing call to review the recording and transcript.
* Admin Usage Monitoring: Internal admins track API costs (OpenAI/Twilio) to ensure we maintain our 80% Gross Margin target.

### AI Insight Generation
```mermaid
sequenceDiagram
    autonumber
    participant Twilio as Twilio (Telephony)
    participant FastAPI
    participant OpenAI as OpenAI (GPT-4o)
    participant DB as Supabase DB

    Note over Twilio, DB: Asynchronous Process Triggered by Call End
    Twilio->>FastAPI: POST /webhook/call-complete (RecordingUrl)
    FastAPI->>FastAPI: Validate X-Twilio-Signature
    FastAPI-->>Twilio: 200 OK (Ack)
    
    rect rgb(240, 248, 255)
        Note right of FastAPI: Background Worker Starts
        FastAPI->>Twilio: GET /transcripts/{CallSid}
        Twilio-->>FastAPI: Return Raw Transcript
        
        FastAPI->>OpenAI: POST /v1/chat/completions (Prompt + Transcript)
        Note right of OpenAI: Analyzes Sentiment, Upsell $, Conversion
        OpenAI-->>FastAPI: Return JSON Insights
        
        FastAPI->>DB: INSERT into 'Insights' table
        DB-->>FastAPI: Confirm Write
    end
```

### Frictionless Onboarding
```mermaid
sequenceDiagram
    autonumber
    actor Manager
    participant Portal as Onboarding Portal
    participant API as Converza API
    participant Twilio as Twilio API
    participant DB as Supabase DB

    Manager->>Portal: Selects Local Area Code
    Portal->>API: POST /provision-number
    API->>Twilio: POST /incoming-phone-numbers (Buy)
    Twilio-->>API: Return New Phone Number
    
    API->>Twilio: POST /configure-webhook (Set Webhook URL)
    Twilio-->>API: Confirmation
    
    API->>DB: UPDATE Tenant (Link New Number)
    DB-->>API: Success
    API-->>Portal: Provisioning Complete
    Portal-->>Manager: Display New Number (Ready for Calls)
```

### Dashboard KPI Retrieval
```mermaid
sequenceDiagram
    autonumber
    actor Manager
    participant Dash as Next.js Dashboard
    participant Auth as Supabase Auth
    participant DB as Supabase DB

    Manager->>Dash: Access "Performance Dashboard"
    Dash->>Auth: Get Current User Session (JWT)
    Auth-->>Dash: Valid Token
    
    Note over Dash, DB: Direct BaaS Interaction via RLS
    Dash->>DB: SELECT avg(conversion_rate), sum(upsell_rev) FROM Insights
    
    DB-->>Dash: Return Aggregated KPIs
    
    Dash-->>Manager: Render ROI Charts & Graphs
```


### Coaching Drill-Down
```mermaid
sequenceDiagram
    autonumber
    actor Manager
    participant Dash as Next.js Dashboard
    participant DB as Supabase DB
    participant Twilio as Twilio Assets

    Manager->>Dash: Click on "Low Performing Call"
    
    par Fetch Metadata
        Dash->>DB: SELECT transcript, sentiment, ai_score FROM Calls WHERE id=X
        DB-->>Dash: Return Insight Data
    and Fetch Audio
        Dash->>Twilio: GET /recordings/{CallSid}.mp3
        Twilio-->>Dash: Stream Audio File
    end
    
    Dash-->>Manager: Display Player + Synchronized Transcript
```
