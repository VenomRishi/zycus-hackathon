# Architecture Decision Records (ADR)

AI Reassignment Engine – Agentic delivery reassignment system for ZipRun.

---

## ADR-1: Where does routing logic live?

**Context**
The system needs to recommend agents for order reassignment. This logic will be called from two places: (1) HTTP endpoints for on-demand suggestions, and (2) async event handlers in the agentic re-planning loop. Routing logic must be decoupled from both call paths.

**Options considered**
- **A) Logic in Controller**: Route recommendation inline in REST endpoint handler. Simple for single use case, but duplicates logic when called from event listener.
- **B) Service Layer (chosen)**: Dedicated `RoutingService` encapsulates routing workflow. Supports both HTTP and async callers without duplication.
- **C) Domain Model**: Move logic into Order/Agent entities as methods. Couples domain model to infrastructure concerns (persistence, strategy selection).

**Decision**
Chose **Option B – Service Layer**. `RoutingService` handles:
1. Active strategy selection from the registry
2. Calling the strategy interface
3. Creating and persisting `ReassignmentSuggestion`
4. Handling fallbacks (AI → rule-based)

Both `OrderController.suggestReassignment()` and `ReassignmentService.onAgentOffline()` call the same service methods, keeping logic centralized.

**Tradeoffs accepted**
Added a service class (minimal complexity cost). No other significant tradeoffs; this is the cleanest separation of concerns.

---

## ADR-2: How does runtime strategy switching work?

**Context**
The system must support multiple routing strategies (rule-based, AI, and sprint 2's zone-affinity). Strategies need to be switchable at runtime via config, with no code restart. Both HTTP endpoints and async event handlers must use the active strategy without modification.

**Options considered**
- **A) Spring @Qualifier + property**: Wire strategy via `@Qualifier("${routing.strategy}")` using an autowired field. Simple but requires restart to swap.
- **B) Auto-wired Map<String, RoutingStrategy> bean map (chosen)**: Spring auto-populates a map keyed by bean name. `RoutingService` reads `routing.strategy` property at call time to select from the map.
- **C) Manual factory with switch statement**: Explicit control, but requires modifying factory code when adding sprint 2 strategies.

**Decision**
Chose **Option B – Bean map with runtime selection**.

Implementation:
```java
// Both strategies are Spring beans with component names
@Component("rule-based") public class RuleBasedStrategy implements RoutingStrategy {}
@Component("ai") public class AIRoutingStrategy implements RoutingStrategy {}

// RoutingService receives the map
private final Map<String, RoutingStrategy> strategies;

// Selection happens at call time
private RoutingStrategy getActiveStrategy() {
  return strategies.get(activeStrategyName); // reads from application.properties
}
```

Switching strategies: change `routing.strategy=ai` to `routing.strategy=rule-based` in `application.properties` (or via environment variable). No restart needed. Sprint 2 adds `@Component("zone-affinity")` and it automatically becomes selectable.

**Tradeoffs accepted**
Less explicit than a switch statement—reader must know Spring auto-populates maps by bean name. Mitigated by clear naming and comments. Runtime failures if config property typo'd (e.g., `routing.strategy=ai-typo`), but we added startup validation logging.

---

## ADR-3: How does the system stay resilient when the LLM is unavailable?

**Context**
LLM calls can fail in multiple ways: timeout, quota exceeded, malformed response, hallucinated agent IDs. The system must stay healthy regardless. This is critical in the async re-planning context: if AI fails, we must still queue a suggestion (rule-based) rather than silently dropping the order.

**Options considered**
- **A) Strict failure**: If LLM fails, throw exception. Bubbles up and crashes re-plan, orders get stuck.
- **B) Silent fallback**: Swallow all exceptions, use rule-based, don't log. Hides failures, hard to debug.
- **C) Explicit fallback with observability (chosen)**: Catch specific failure modes, log them, fall back to rule-based, return the fallback suggestion.

**Decision**
Chose **Option C**.

Implementation in `AIRoutingStrategy`:
1. **Timeout/Connection errors**: Catch `RestClientException`, log warning, fall back
2. **Malformed JSON**: Catch `JsonProcessingException`, log error, fall back
3. **Hallucinated agent IDs**: Parse JSON, validate agent exists in roster before accepting. If invalid, throw and trigger fallback
4. **Confidence out of range**: Clamp to [0.0, 1.0] with warning log

In both prompt contexts (initial and re-plan), if `AIRoutingStrategy` fails, `RoutingService.suggestReplan()` catches and calls the rule-based strategy as backup. Suggestion is persisted with confidence=0.8 (rule-based marker) so ops knows it's a fallback.

Example flow:
- Agent goes offline
- Async re-plan triggered for affected order
- AI strategy called, but LLM times out
- Exception caught in `AIRoutingStrategy.recommendAgentForReplan()`
- Falls back to rule-based strategy
- Suggestion persisted with rule-based recommendation and logged
- Order transitions to REASSIGNMENT_PENDING
- Ops sees suggestion with lower confidence (indicates fallback was used)

**Tradeoffs accepted**
More error handling code, but necessary for agentic loops. No silent failures means diagnostics are visible in logs. Trade-off is worth it for system reliability.

---

## ADR-4: How is the agentic loop triggered and kept off the request path?

**Context**
When an agent goes OFFLINE, the system must identify affected orders, generate re-plan suggestions, and queue them for ops—all asynchronously so the status update endpoint returns immediately. The loop must be event-driven (reactive to state change) not scheduled (polling).

**Options considered**
- **A) Scheduled poller**: Every 5 minutes, check for OFFLINE agents. Wrong model—not reactive, wastes compute, misses immediate context.
- **B) CompletableFuture in controller**: Wrap re-planning in `CompletableFuture.runAsync()` inline. Works, but couples endpoint to re-plan logic.
- **C) Spring @EventListener + @Async thread pool (chosen)**: Publish `AgentOfflineEvent` from status update endpoint. `@EventListener` method runs async. Clean separation of concerns.

**Decision**
Chose **Option C**.

Implementation flow:
1. **PATCH /agents/{id}/status** endpoint updates agent status and publishes event:
   ```java
   if (newStatus == AgentStatus.OFFLINE) {
       eventPublisher.publishEvent(new AgentOfflineEvent(this, agentId));
   }
   return ResponseEntity.ok(agent); // returns immediately
   ```

2. **ReassignmentService** listens for the event:
   ```java
   @EventListener
   @Async("replanExecutor")  // named thread pool
   public void onAgentOffline(AgentOfflineEvent event) {
       // identify affected orders
       // run routing for each
       // persist suggestions
   }
   ```

3. **AsyncConfig** defines thread pool:
   ```java
   @Bean("replanExecutor")
   public Executor replanExecutor() {
       // core=2, max=4, queue=100
   }
   ```

Why Spring's event system:
- Clean decoupling: endpoint doesn't know about re-planning
- Works with both HTTP and async contexts naturally
- Exception handling per task (doesn't bubble to HTTP)
- Observable: logs can track re-plan task completion

**Tradeoffs accepted**
Async tasks run outside HTTP request lifecycle—exceptions don't bubble to client. Mitigated by explicit logging: all failures are logged with context. Must test that async tasks complete before demo. Potential for future missed edge case if re-plan task hangs (application won't know), but startup validation and explicit task timeout help.

---

## ADR-5: What did you design to extend, and what did you save for later?

### Extension Seams (Ready for Sprint 2/3)

**Domain Model**
- `Order` has nullable `pickupZone`, `dropoffZone`, `weightClass`, `slaDeadline` fields
- `Agent` has nullable `currentZone`, `maxCapacity` fields
- No schema migration needed: existing rows get NULL, new rows populate the fields
- **Extension**: Sprint 2 adds zone-affinity logic; Sprint 3 adds SLA proactive checks

**Routing Strategy Interface**
```java
public interface RoutingStrategy {
  RecommendedAgent recommendAgent(Order order, List<Agent> availableAgents);
}
```
- New strategies implement this interface and register as Spring beans
- **Extension**: `ZoneAffinityStrategy` implements this interface and gets added as `@Component("zone-affinity")`
- **Extension**: `TrafficAwareStrategy` uses AI with tools to call traffic API
- No existing code changes needed; just implement interface + register bean

**Agentic Loop Trigger**
- Currently fires on OFFLINE event
- Pattern is generic: publish event, @EventListener responds
- **Extension**: Spring 3 adds SLA proactive check: publish `OrderSLABreachEvent`, same `@EventListener` pattern
- No re-architecting needed

### Deliberate Exclusions (Priority Rationale)

**1. Full Dispatch Board**
Status: **Deferred** (would be UI ceiling)
Reasoning: The agentic loop is a correctness requirement (agent fails → orders get stuck without it). The full dispatch board showing all orders across all statuses is a visibility enhancement, not a blocker. Prioritized correctness over coverage.

**2. SLA Countdown Timers**
Status: **Deferred** (UI enhancement for Sprint 3)
Reasoning: Current UI shows suggestion + accept/reject, which unblocks ops. SLA countdown adds time-pressure feedback but isn't needed for the re-plan path to work.

**3. Server-Sent Events (SSE) Token Streaming**
Status: **Bonus** (not shipped)
Reasoning: Would show AI reasoning tokens arriving in real-time. Nice-to-have for UX, not required for functionality. Ops can see final reasoning in card details.

**4. Persistent Database**
Status: **Deferred** (used H2 in-memory)
Reasoning: In-memory H2 is sufficient for demo. Sprint 2 can add Postgres for persistence. No architectural coupling to H2; just change `application.properties` datasource URL.

**5. Agent Zone Map Visualization**
Status: **Deferred** (UI enhancement)
Reasoning: Would display zones geographically. Current roster shows zone text. Not needed for core reassignment loop.

---

## Summary

This design prioritizes:
1. **Correctness over completeness**: Agentic loop works end-to-end before UI polish
2. **Decoupling**: Strategies, services, and events are loosely coupled
3. **Observability**: All failures logged with context
4. **Extensibility**: Sprint 2/3 additions are drop-ins, not rewrites
5. **Resilience**: AI unavailability doesn't break the system

---

## Build & Run

### Backend
```bash
cd backend
mvn clean install
mvn spring-boot:run
```

### Frontend
```bash
cd frontend
npm install
npm run dev
```

Visit **http://localhost:5173** to see the ops dashboard.

Test the agentic loop:
1. Visit `http://localhost:8080/h2-console` to see seed data
2. In the ops dashboard, find an agent and change status to OFFLINE
3. Watch suggestions appear automatically within 3 seconds (auto-refresh)
4. Accept/reject suggestions

---

**Walkthrough Topics**:
- Trace an order through the domain model
- Walk the agentic loop: agent OFFLINE → re-plan fire → suggestion queue → ops action
- Explain strategy switching mechanism
- Show AI strategy with two distinct prompts (initial vs re-plan)
- Discuss fallback behavior when LLM unavailable
