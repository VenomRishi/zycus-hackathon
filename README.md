# AI Reassignment Engine

An agentic delivery reassignment system that automatically detects when agents go offline, identifies affected orders, uses AI to recommend optimal reassignments, and surfaces suggestions to ops for approval.

## 🚀 Quick Start (< 5 minutes)

### Prerequisites
- Java 17+
- Maven 3.6+
- Node.js 16+
- npm

### 1. Start Backend
```bash
cd backend

# Set LLM credentials (required for AI routing)
export LLM_CLIENT_ID="b9213bc6-dee4-480e-9d8a-861f52e81ab1"
export LLM_CLIENT_SECRET="Jkr8Q~oznuPHLAYkOikyTIBLVp6qpafp1.yxxdBt"
export LLM_TENANT_ID="bfa3dfb0-91d5-4bf7-9a0c-fbf6ff337187"

# Build & run
mvn clean install
mvn spring-boot:run
```

Backend runs on **http://localhost:8080**

### 2. Start Frontend
```bash
cd frontend
npm install
npm run dev
```

Frontend runs on **http://localhost:5173**

### 3. Test the Agentic Loop

**Option A: Via Frontend**
1. Open http://localhost:5173
2. Go to "Agent Roster" tab
3. Find any agent and change status to **OFFLINE** using the dropdown
4. Switch to "Reassignments" tab
5. Within 3 seconds, you'll see a new suggestion with **⚡ Re-plan** badge
6. Click "Accept & Reassign" to approve

**Option B: Via curl**
```bash
# List agents
curl http://localhost:8080/agents

# Set agent to OFFLINE (triggers agentic re-plan)
curl -X PATCH http://localhost:8080/agents/AGT-001/status \
  -H "Content-Type: application/json" \
  -d '{"status":"OFFLINE"}'

# List pending suggestions (should see new AGENT_OFFLINE suggestions)
curl http://localhost:8080/suggestions?status=PENDING

# Accept a suggestion
curl -X PATCH http://localhost:8080/suggestions/SUGGESTION_ID \
  -H "Content-Type: application/json" \
  -d '{"status":"ACCEPTED"}'
```

---

## 📋 What's Included

### Backend (Spring Boot 3.x)
- **Domain Model**: Order, Agent, ReassignmentSuggestion entities with clean state machines
- **Routing Engine**: Pluggable strategy pattern (rule-based + AI)
  - **RuleBasedStrategy**: Assigns to agent with fewest orders
  - **AIRoutingStrategy**: Uses internal LLM to recommend reassignment
  - Runtime switchable via config, no restart
- **LLM Integration**: Internal Godrej Capital LLM service with fallback handling
  - Two distinct prompts: initial assignment vs. re-plan after agent offline
  - Validates hallucinated agent IDs
  - Falls back to rule-based if AI unavailable
- **Agentic Loop**: Event-driven async re-planning when agent goes OFFLINE
  - Identifies affected orders
  - Queues reassignment suggestions for ops approval
  - Human checkpoint preserved (system proposes, ops approves)
- **REST API**: 
  - POST /orders – Create order
  - GET /orders?status=... – List orders
  - PATCH /agents/{id}/status – Update agent (fires agentic loop)
  - GET /suggestions?status=PENDING – List pending suggestions
  - PATCH /suggestions/{id} – Accept/reject suggestion

### Frontend (React 18 + Vite)
- **Reassignments Tab**: Pending suggestions with inline AI reasoning, confidence scores, accept/reject controls
  - **Re-plan Badge**: Visual indicator (⚡) for automatic re-planning after agent OFFLINE
- **Agent Roster Tab**: All agents with status, active orders, zone, capacity
  - Status dropdown to change agent availability
- **Auto-Refresh**: Polls backend every 3 seconds for new suggestions
- **Loading States & Error Handling**: User-friendly feedback

---

## 🏗️ Architecture

### Agentic Loop Flow
```
[Agent goes OFFLINE]
        ↓
[AgentOfflineEvent published]
        ↓
[@EventListener fires @Async]
        ↓
[Identify affected ASSIGNED orders]
        ↓
[Run active routing strategy on each]
        ↓
[Create ReassignmentSuggestion (triggerReason=AGENT_OFFLINE)]
        ↓
[Persist suggestions, transition orders to REASSIGNMENT_PENDING]
        ↓
[Ops sees suggestions in UI with ⚡ Re-plan badge]
        ↓
[Ops accepts/rejects → suggestion status updated]
```

### Strategy Switching
Strategies registered as Spring beans:
- `@Component("rule-based")` RuleBasedStrategy
- `@Component("ai")` AIRoutingStrategy

`RoutingService` selects at runtime based on `routing.strategy` property:
```properties
routing.strategy=ai  # or rule-based
```

Change in application.properties or environment variable; no restart needed.

### Fallback Resilience
```
AI Strategy Called
    ↓
LLM Request → [timeout|error|invalid response]
    ↓
Exception caught, logged
    ↓
Fall back to Rule-Based Strategy
    ↓
Persist suggestion with confidence=0.8 (fallback marker)
```

---

## 📁 Project Structure

```
zycus-hackathon/
├── backend/
│   ├── pom.xml
│   ├── src/main/java/com/ziprun/reassignment/
│   │   ├── ReassignmentEngineApplication.java
│   │   ├── entity/               # JPA entities: Agent, Order, ReassignmentSuggestion
│   │   ├── repository/           # Spring Data repositories
│   │   ├── service/              # Business logic
│   │   │   ├── RoutingStrategy.java       # Interface
│   │   │   ├── RuleBasedStrategy.java
│   │   │   ├── AIRoutingStrategy.java
│   │   │   ├── RoutingService.java        # Orchestrates routing
│   │   │   ├── ReassignmentService.java   # Async re-plan listener
│   │   │   ├── AgentService.java
│   │   │   ├── OrderService.java
│   │   │   └── LLMGateway.java            # LLM client
│   │   ├── controller/           # REST endpoints
│   │   ├── event/                # Domain events
│   │   ├── config/               # Async executor config
│   │   └── dto/                  # Request/response DTOs
│   ├── src/main/resources/
│   │   ├── application.properties
│   │   └── data.sql              # Seed data (5 agents, 8 orders)
│   └── README.md
├── frontend/
│   ├── package.json
│   ├── vite.config.js
│   ├── index.html
│   ├── src/
│   │   ├── main.jsx
│   │   ├── App.jsx
│   │   ├── components/           # React components
│   │   ├── services/
│   │   │   └── api.js            # Backend API client
│   │   └── styles/               # CSS
│   └── README.md
├── ADR.md                        # Architecture Decision Records
└── README.md                     # This file
```

---

## 🔧 Configuration

### LLM Service (Internal Godrej Capital)
Required environment variables:
```bash
export LLM_CLIENT_ID="b9213bc6-dee4-480e-9d8a-861f52e81ab1"
export LLM_CLIENT_SECRET="Jkr8Q~oznuPHLAYkOikyTIBLVp6qpafp1.yxxdBt"
export LLM_TENANT_ID="bfa3dfb0-91d5-4bf7-9a0c-fbf6ff337187"
```

### Routing Strategy
In `backend/src/main/resources/application.properties`:
```properties
routing.strategy=rule-based  # or "ai"
```

### Database
H2 in-memory (auto-initialized):
```properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.h2.console.enabled=true  # Access at http://localhost:8080/h2-console
```

---

## 📊 Seed Data

On startup, the backend loads:
- **5 Agents**: Mix of AVAILABLE, BUSY statuses with order load
- **8 Orders**: All ASSIGNED to agents

See `backend/src/main/resources/data.sql`.

---

## 🧪 API Endpoints

### Orders
- `POST /orders` – Create order
- `GET /orders?status=ASSIGNED|REASSIGNMENT_PENDING|REASSIGNED|DELIVERED` – List
- `GET /orders/{id}` – Get details
- `POST /orders/{id}/suggest` – Request suggestion

### Agents
- `GET /agents` – List all
- `GET /agents/{id}` – Get details
- `GET /agents/available` – List available only
- `PATCH /agents/{id}/status` – Update status (fires agentic loop if OFFLINE)

### Suggestions
- `GET /suggestions?status=PENDING|ACCEPTED|REJECTED` – List
- `GET /suggestions/{id}` – Get details
- `PATCH /suggestions/{id}` – Update status (accept/reject)

---

## 📝 ADR & Documentation

See [ADR.md](./ADR.md) for detailed architectural decisions:
1. **ADR-1**: Where routing logic lives (Service Layer)
2. **ADR-2**: How runtime strategy switching works (Bean map + config)
3. **ADR-3**: LLM resilience & fallback handling
4. **ADR-4**: Agentic loop trigger mechanism (@EventListener + @Async)
5. **ADR-5**: Extension seams for Sprint 2/3 and deliberate exclusions

---

## 🛣️ Roadmap

### Sprint 2: Zone & Capacity
- Zone-aware routing strategy
- Agent max capacity constraints
- Order weight classes (LIGHT/HEAVY)

### Sprint 3: Proactive & SLA
- SLA deadline monitoring
- Proactive re-planning before breach
- Priority tier support

---

## 🧠 Key Design Decisions

| Decision | Approach | Why |
|----------|----------|-----|
| Routing Pattern | Service Layer + Strategy pattern | Decouples from both HTTP and async callers |
| Strategy Switchability | Bean map + config | No restart to switch strategies; scales for Sprint 2 |
| LLM Resilience | Explicit catch + fallback | AI unavailability doesn't crash the system |
| Agentic Loop | @EventListener + @Async | Event-driven, reactive, keeps main thread unblocked |
| Human Checkpoint | Suggestion queue + ops approval | System proposes, ops disposes (intentional) |

---

## 🐛 Troubleshooting

**Backend won't start?**
- Verify Java 17+ installed: `java -version`
- Check Maven: `mvn -version`
- Try `mvn clean install -U` to refresh dependencies

**Frontend shows CORS errors?**
- Ensure backend is running on http://localhost:8080
- Verify `backend/src/main/resources/application.properties` has CORS enabled for localhost:5173

**LLM calls timing out?**
- Check network access to https://uat-saksham.godrejcapital.com
- Verify credentials are set correctly
- System falls back to rule-based routing automatically; check logs

**Suggestions not appearing after agent goes OFFLINE?**
- Check backend logs for async re-plan task
- Verify agent status change was successful (check roster)
- Wait 3 seconds for auto-refresh or click manual refresh

**Port conflicts?**
- Backend: Change `server.port` in `application.properties`
- Frontend: `npm run dev -- --port 5174`

---

## 📹 Demo Flow

1. **Setup**: Start backend + frontend
2. **Show seed data**: List agents and orders in roster/list views
3. **Trigger agentic loop**: 
   - Select an agent
   - Change status to OFFLINE
   - Show re-plan suggestions appear (⚡ badge)
4. **Show AI reasoning**: 
   - Expand suggestion details
   - Show confidence score
   - Show AI-generated reasoning text
5. **Approval flow**: 
   - Accept a suggestion
   - Show order transitions to REASSIGNED
   - Show agent status reflected in roster
6. **Fallback resilience** (optional):
   - Stop LLM or change credentials
   - Trigger agent OFFLINE again
   - Show suggestions still appear (rule-based with lower confidence)

---

## 📞 Support

For questions or issues, refer to:
- Backend README: [backend/README.md](./backend/README.md)
- Frontend README: [frontend/README.md](./frontend/README.md)
- ADR & Architecture: [ADR.md](./ADR.md)

---

**Built for ZipRun | Godrej Capital AI Hackathon**
