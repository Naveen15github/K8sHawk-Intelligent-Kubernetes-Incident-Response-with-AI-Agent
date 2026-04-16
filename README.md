# 🦅 K8sHawk — Intelligent Kubernetes Incident Response with AI Agent

> **Note:** Unfortunately, the source code files for this project were lost during development. This README documents the complete system as it was built and demonstrated. The architecture, technical decisions, and full working demo are captured here for reference and future reconstruction.

---

## 📐 Architecture Diagram

![Architecture diagram]()


---

## 🔍 What is K8sHawk?

**K8sHawk** is an end-to-end autonomous Kubernetes incident response system that I built to eliminate the manual toil of debugging Kubernetes cluster failures. When a pod crashes, fails to pull an image, runs out of memory, or hits any other common Kubernetes failure, K8sHawk automatically:

1. **Detects** the incident in real time by watching the Kubernetes event stream
2. **Investigates** the root cause using an AI agent (Groq LLM) that runs actual `kubectl` commands iteratively — just like a human SRE would
3. **Generates a voice alert** in English, Tamil, or Tanglish using Sarvam AI's TTS API, and uploads the audio directly to Slack
4. **Sends a rich Slack notification** with full RCA, severity rating, and an interactive one-click fix button
5. **Executes the approved fix** command (`kubectl delete`, `kubectl rollout restart`, etc.) when you click Approve in Slack
6. **Monitors recovery** by polling the cluster every 15 seconds and reports back to the Slack thread
7. **Saves documentation** as structured Markdown RCA files, organized by date

The system handles everything from image pull failures and crash loops to OOMKilled events and scheduling failures — all with zero human investigation required.

---

## ✨ Key Features

- **Agentic AI Investigation** — The LLM doesn't just read static data. It iteratively calls `kubectl get`, `kubectl describe`, `kubectl logs`, and `kubectl top` in a tool-use loop (up to 5 rounds) to investigate the incident like a real engineer
- **Multi-model Fallback** — Primary: `llama-3.3-70b-versatile`. Falls back silently to `llama-3.1-8b-instant` on rate limits, then to pattern-based kubectl analysis if both are unavailable
- **Voice Alerts (Multilingual)** — Generates spoken summaries in English (Priya), Tamil (Anushka), or Tanglish via Sarvam AI TTS and uploads the MP3 directly to the Slack thread
- **Interactive Slack Notifications** — Rich Block Kit messages with severity colors, RCA summaries, kubectl commands, and Approve / Dismiss buttons
- **One-Click Fix Execution** — Clicking Approve in Slack triggers the fix command on the actual cluster with real-time terminal output and Slack thread updates
- **Recovery Monitoring** — Polls cluster health every 15 seconds for up to 3 minutes after fix execution
- **Deduplication** — Prevents duplicate alerts for the same pod within a 60-second window
- **Date-Organized Storage** — RCA Markdown files and voice MP3s are saved under `knowledge-base/` organized by date

---

## 🛠️ Technology Stack

| Layer | Technology |
|---|---|
| Language | Python 3.11+ |
| API Server | FastAPI + Uvicorn |
| Async Runtime | asyncio |
| Kubernetes Client | kubernetes-python |
| AI Investigation | Groq API (`llama-3.3-70b-versatile`) |
| Voice Generation | Sarvam AI TTS |
| Notifications | Slack SDK (Block Kit) |
| HTTP Client | httpx (async) |
| Config Management | Pydantic Settings |
| Terminal UI | Rich |
| Local Cluster | k3d (k3s in Docker) |

---

## 📸 System Walkthrough — Live Demo

The following screenshots show the complete K8sHawk pipeline end-to-end, from cluster setup to incident resolution. Each step is shown in sequence as it happened during a live demo run.

---

### Step 1 — Kubernetes Cluster Running via Docker Desktop (k3d)

The cluster backing K8sHawk is a lightweight k3d cluster (k3s running inside Docker containers) managed via Docker Desktop. The five containers shown represent the k3d loadbalancer, server node, and three agent nodes.

![Docker Desktop - k3d cluster containers](https://github.com/Naveen15github/K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent/blob/508904254f49329871326cd00df33c1579f0f220/Screenshot%20(584).png?raw=true)

---

### Step 2 — K8sHawk Starts Up and Connects to the Cluster

Running `python -m k8shawk.main` launches the application. It loads configuration, initializes all services, connects to the Kubernetes cluster, prints a live summary of cluster stats (3 nodes, 11 pods, 5 namespaces, 7 services, 6 deployments), and begins watching all namespaces for Warning events.

![K8sHawk startup banner and cluster status](https://github.com/Naveen15github/K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent/blob/508904254f49329871326cd00df33c1579f0f220/Screenshot%20(573).png?raw=true)

---

### Step 3 — Triggering a Real Incident (Bad Image Pull)

To simulate a real incident, I created a pod using a nonexistent Docker image. This causes Kubernetes to emit a `Failed` / `ImagePullBackOff` warning event, which K8sHawk detects immediately. The second command shows it already exists from a previous test run.

![Creating test pod with bad image in PowerShell](https://github.com/Naveen15github/K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent/blob/508904254f49329871326cd00df33c1579f0f220/Screenshot%20(582).png?raw=true)

---

### Step 4 — Incident Detected, AI Investigation Begins

K8sHawk picks up the `Failed` event on `test-incident` in the `default` namespace within seconds. The full error message from the Kubernetes event is captured. The AI investigator starts working — it first tries the primary Groq model, hits a rate limit, silently switches to the fallback model, which is also rate-limited, and finally falls back to kubectl-based pattern analysis. The voice alert generation then begins by calling the Sarvam TTS API.

![Incident detected and AI pipeline running](https://github.com/Naveen15github/K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent/blob/508904254f49329871326cd00df33c1579f0f220/Screenshot%20(574).png?raw=true)

---

### Step 5 — Voice Alert Generated, Uploaded to Slack, RCA Saved

The Sarvam TTS API generates a 1.8MB MP3 voice alert file. It is uploaded to Slack using the `files.uploadV2` workflow and attached to the incident thread. The RCA Markdown file is simultaneously saved to `knowledge-base/rca/2026-04-16/`. The incident response pipeline completes in full with severity MEDIUM.

![Voice upload, Slack confirmation, RCA save](https://github.com/Naveen15github/K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent/blob/508904254f49329871326cd00df33c1579f0f220/Screenshot%20(575).png?raw=true)

---

### Step 6 — Rich Alert Arrives in Slack with Approve / Dismiss Buttons

The Slack notification arrives in `#k8shawk-alerts` with full Block Kit formatting. It shows the severity level (🟡 MEDIUM), the affected pod and namespace, detection timestamp, RCA summary, proposed kubectl fix command, and two interactive buttons — **✅ Approve Fix** and **❌ Dismiss**. The alert is structured so any on-call engineer can understand the situation without touching the terminal.

![Slack alert with severity, RCA, and action buttons](https://github.com/Naveen15github/K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent/blob/508904254f49329871326cd00df33c1579f0f220/Screenshot%20(577).png?raw=true)

---

### Step 7 — Slack Thread Shows Full RCA and Voice Alert Audio Player

The Slack thread reply includes the complete Root Cause Analysis text — the full investigation output written by the AI explaining what happened, why, and what to review. Below it, the uploaded MP3 audio file appears as an inline waveform player labelled **K8sHawk Voice Alert** (0:41 duration, 2 MB), so the engineer can listen to the spoken summary without reading.

![Slack thread with full RCA text and voice audio player](https://github.com/Naveen15github/K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent/blob/508904254f49329871326cd00df33c1579f0f220/Screenshot%20(576).png?raw=true)

---

### Step 8 — Engineer Clicks Approve Fix — Confirmation Dialog Appears

When the on-call engineer clicks **✅ Approve Fix** in Slack, a confirmation modal appears showing exactly which command will be run on the cluster: `kubectl describe pod test-incident -n default`. This gives one final chance to review before execution.

![Slack confirmation dialog before fix execution](https://github.com/Naveen15github/K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent/blob/508904254f49329871326cd00df33c1579f0f220/Screenshot%20(578).png?raw=true)

---

### Step 9 — Fix Approved in Terminal, Execution Starts

The Slack button click is received by the FastAPI webhook server. The server logs show the full request trace with a unique request ID (`REQ-1912ab91`), action ID, and thread timestamp. The fix command is confirmed, the background task is started, and the terminal shows the complete fix execution box — pod name, namespace, command, and description.

![Terminal showing fix approval and execution kickoff](https://github.com/Naveen15github/K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent/blob/508904254f49329871326cd00df33c1579f0f220/Screenshot%20(581).png?raw=true)

---

### Step 10 — kubectl Command Runs, Fix Applied Successfully

The kubectl command executes with a 30-second subprocess timeout. The full command output is shown — the pod's name, namespace, priority, service account, node assignment, start time, labels, annotations, status, and IP address. K8sHawk confirms the fix succeeded and begins monitoring recovery every 15 seconds.

![Terminal showing kubectl output and Fix Applied Successfully banner](https://github.com/Naveen15github/K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent/blob/508904254f49329871326cd00df33c1579f0f220/Screenshot%20(580).png?raw=true)

---

### Step 11 — Fix Results Posted Back to Slack Thread

The fix command output is immediately posted back to the Slack thread so the engineer sees the result without switching context. The full `kubectl describe` output — including node assignment, container image (`nonexistent-image:latest`), and pod status (`Pending`) — appears inline in the thread alongside the **Applying fix** status message.

![Slack thread showing applied fix output](https://github.com/Naveen15github/K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent/blob/508904254f49329871326cd00df33c1579f0f220/Screenshot%20(579).png?raw=true)

---

## 📁 Project Structure

```
k8shawk/
├── main.py                          # Application entry point
├── config/
│   └── settings.py                  # Pydantic settings (env vars, defaults)
├── models/
│   ├── incident.py                  # KubernetesIncident dataclass
│   ├── analysis.py                  # IncidentAnalysis + Severity enum
│   └── fix.py                       # PendingFix dataclass
├── services/
│   ├── event_watcher.py             # K8s event stream watcher + deduplication
│   ├── ai_investigator.py           # Groq agentic tool loop (kubectl tools)
│   ├── voice_generator.py           # Sarvam AI TTS + file storage
│   ├── notification_service.py      # Slack Block Kit messages + audio upload
│   ├── fix_executor.py              # kubectl fix execution + recovery monitor
│   ├── rca_manager.py               # Markdown RCA file writer
│   └── kubectl_tools.py             # Safe kubectl subprocess wrapper
├── handlers/
│   └── incident_handler.py          # Orchestrates the full response pipeline
└── server/
    └── webhook_server.py            # FastAPI webhook for Slack button interactions

knowledge-base/
├── rca/
│   └── YYYY-MM-DD/
│       └── rca_{pod}_{timestamp}.md
└── voice-messages/
    └── YYYY-MM-DD/
        └── voice_{pod}_{timestamp}.mp3

k8s/
└── k8shawk-deployment.yaml          # Kubernetes manifests (ServiceAccount, RBAC, Deployment)
```

> ⚠️ **Note:** The source code files listed above were accidentally lost. The structure and implementation details documented here are accurate to the working version demonstrated in the screenshots above.

---

## ⚙️ Configuration

All configuration is loaded from a `.env` file using Pydantic Settings.

```bash
# .env

# API Keys
GROQ_API_KEY=your_groq_api_key_here
SLACK_BOT_TOKEN=xoxb-your-slack-bot-token
SARVAM_API_KEY=your_sarvam_api_key_here

# Slack
SLACK_CHANNEL=#k8shawk-alerts

# Voice
VOICE_LANGUAGE=english        # Options: english | tamil | tanglish

# Webhook
WEBHOOK_PORT=8081

# Kubernetes
WATCH_NAMESPACE=              # Leave empty to watch all namespaces

# Recovery
RECOVERY_TIMEOUT_SECONDS=180
```

---

## 🚀 How to Run

### Prerequisites

- Python 3.11+
- A running Kubernetes cluster (local k3d, minikube, or remote)
- `kubectl` configured and pointing to your cluster
- ngrok (to expose the webhook server to Slack)
- Slack app with bot token and interactive components enabled

### Setup

```bash
# Clone the repository
git clone https://github.com/Naveen15github/K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent.git
cd K8sHawk-Intelligent-Kubernetes-Incident-Response-with-AI-Agent

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env with your API keys

# Expose webhook for Slack (in a separate terminal)
ngrok http 8081
# Copy the ngrok HTTPS URL → paste into Slack app's Interactivity & Shortcuts URL
# Format: https://your-ngrok-url.ngrok.io/slack/interactions

# Start K8sHawk
python -m k8shawk.main
```

### Triggering a Test Incident

```bash
# Create a pod with a nonexistent image — triggers ImagePullBackOff / Failed event
kubectl run test-incident --image=nonexistent-image:latest

# Clean up after testing
kubectl delete pod test-incident
```

### Verify Webhook is Working

```bash
curl http://localhost:8081/health
# → {"status": "healthy"}

curl http://localhost:8081/slack/validate
# → Returns ngrok URL and server config info
```

---

## 🔄 Incident Response Pipeline

```
K8s Warning Event
       │
       ▼
 EventWatcher detects it
 (deduplication: 60s window per pod)
       │
       ▼
 IncidentHandler orchestrates:
       │
       ├─ 1. AIInvestigator.investigate()
       │       └─ Agentic tool loop:
       │           kubectl get → kubectl describe → kubectl logs → kubectl top
       │           (up to 5 rounds, primary → fallback → basic analysis)
       │
       ├─ 2. VoiceGenerator.generate_alert()
       │       └─ Sarvam TTS API → MP3 saved to knowledge-base/
       │
       ├─ 3. NotificationService.post_incident_alert()
       │       └─ Slack Block Kit → Audio upload → Approve/Dismiss buttons
       │
       ├─ 4. RCAManager.save_rca()
       │       └─ Markdown file → knowledge-base/rca/YYYY-MM-DD/
       │
       └─ 5. Wait for Slack button click...
               │
               ├─ Approve → FixExecutor.apply_fix()
               │               └─ kubectl command → recovery polling → Slack update
               │
               └─ Dismiss → Log dismissal, no action
```

---

## 🧠 AI Investigation — How It Works

The `AIInvestigator` runs an agentic tool loop where the LLM is given access to four kubectl tools:

| Tool | Command |
|---|---|
| `kubectl_get` | `kubectl get <resource> -n <namespace>` |
| `kubectl_describe` | `kubectl describe <resource> <name> -n <namespace>` |
| `kubectl_logs` | `kubectl logs <pod> -n <namespace> --tail=50` |
| `kubectl_top` | `kubectl top pods -n <namespace>` |

The model receives the incident details and the full cluster context prompt, then decides which commands to run. It iterates up to 5 rounds, reads the outputs, and produces a structured JSON response containing severity, severity reason, full RCA (3–5 sentences), one-line RCA summary, fix command, and fix description.

If the primary model (`llama-3.3-70b-versatile`) hits a rate limit, it silently switches to `llama-3.1-8b-instant`. If that also fails, it generates a basic analysis from kubectl pattern matching (CrashLoopBackOff → restart pod, ImagePullBackOff → delete pod to stop retries, OOMKilled → review memory limits, etc.).

---

## 📊 Severity Levels

| Severity | Color | Indicator | Examples |
|---|---|---|---|
| LOW | 🟢 Green | `#36a64f` | Minor warnings, non-critical events |
| MEDIUM | 🟡 Yellow | `#ffcc00` | ImagePullBackOff, image not found |
| HIGH | 🟠 Orange | `#ff8c00` | CrashLoopBackOff, OOMKilled |
| CRITICAL | 🔴 Red | `#cc0000` | Multiple pods down, node failures |

---

## 🎙️ Voice Alert Languages

| Setting | Language | Speaker | Voice ID |
|---|---|---|---|
| `english` | English (India) | Priya (Female) | `en-IN` |
| `tamil` | Tamil | Anushka (Female) | `ta-IN` |
| `tanglish` | Tamil + English mix | Anushka (Female) | `ta-IN` |

Voice scripts are capped at 500 characters due to the Sarvam API constraint and are automatically truncated if needed. Generated MP3 files are stored in `knowledge-base/voice-messages/YYYY-MM-DD/` and deleted locally after successful Slack upload.

---

## 🔐 Kubernetes RBAC Requirements

For in-cluster deployment, K8sHawk needs the following permissions via ClusterRole:

```yaml
- pods:         get, list, watch, delete
- events:       get, list, watch
- deployments:  get, list, watch, patch, update
- nodes:        get, list, watch
- namespaces:   get, list, watch
- services:     get, list, watch
- configmaps:   get, list, watch
```

---

## 📈 Performance Characteristics

| Stage | Typical Duration |
|---|---|
| Event detection latency | < 5 seconds |
| AI investigation | 10–30 seconds |
| Voice generation | 2–5 seconds |
| Slack notification | 1–3 seconds |
| Fix execution | 1–5 seconds |
| Recovery monitoring | Up to 3 minutes (15s polling) |
| **Total pipeline (excl. recovery)** | **15–45 seconds** |

---

## 🏗️ In-Cluster Deployment

K8sHawk can be deployed inside the cluster it monitors using the provided Kubernetes manifests:

```bash
# Apply all manifests
kubectl apply -f k8s/k8shawk-deployment.yaml

# The manifest creates:
# - ServiceAccount: k8shawk
# - ClusterRole + ClusterRoleBinding (RBAC)
# - Secret (API keys)
# - Deployment (1 replica, python -m k8shawk.main)
# - Service (ClusterIP, port 8081)
```

---

## 🔭 What I Learned

Building K8sHawk gave me hands-on experience with:

- **Agentic AI systems** — designing tool-use loops where an LLM drives investigation decisions iteratively rather than receiving a static prompt
- **Kubernetes internals** — working with the event stream API, pod lifecycle states, and RBAC in a real cluster
- **Multi-service orchestration** — wiring together Groq, Sarvam, Slack SDK, FastAPI, and the Kubernetes Python client into a cohesive async pipeline
- **Production-grade error handling** — graceful degradation across model fallbacks, silent voice failures, deduplication, and subprocess timeouts
- **Slack Block Kit** — building interactive messages with buttons, file uploads via the V2 upload flow, and thread-based conversations
- **k3d for local clusters** — running a full multi-node Kubernetes environment inside Docker for rapid local testing

---

## 📝 License

This project is for educational and demonstration purposes.

---

*Built by Naveen G — K8sHawk 🦅 | Intelligent Kubernetes Incident Response*
