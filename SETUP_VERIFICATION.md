# Setup Verification Checklist

Run through this checklist to ensure everything is working before testing the agentic loop.

## 🔍 Pre-Flight Checks

### System Requirements
- [ ] Java 17+ installed: `java -version`
- [ ] Maven 3.6+ installed: `mvn -version`
- [ ] Node.js 16+ installed: `node -v`
- [ ] npm 7+ installed: `npm -v`

### Environment Variables
- [ ] Export LLM credentials:
  ```bash
  export LLM_CLIENT_ID="b9213bc6-dee4-480e-9d8a-861f52e81ab1"
  export LLM_CLIENT_SECRET="Jkr8Q~oznuPHLAYkOikyTIBLVp6qpafp1.yxxdBt"
  export LLM_TENANT_ID="bfa3dfb0-91d5-4bf7-9a0c-fbf6ff337187"
  ```

## 🏗️ Backend Setup

### Build
```bash
cd backend
mvn clean install
```
- [ ] Build completes successfully
- [ ] No dependency errors
- [ ] `target/` directory created

### Run
```bash
mvn spring-boot:run
```
- [ ] App starts without errors
- [ ] Logs show "Tomcat started on port 8080"
- [ ] Logs show "Replan executor initialized"
- [ ] No connection errors to LLM service (can be ignored if credentials are invalid)

### Seed Data
```bash
curl http://localhost:8080/agents
```
- [ ] Returns 5 agents (AGT-001 through AGT-005)
- [ ] Agents have status, active order counts, zones

```bash
curl http://localhost:8080/orders
```
- [ ] Returns 8 orders (ORD-001 through ORD-008)
- [ ] All orders have status ASSIGNED
- [ ] Each order assigned to one of the 5 agents

### H2 Console (Optional)
```
http://localhost:8080/h2-console
```
- [ ] Open in browser
- [ ] Login with sa / (empty password)
- [ ] Select `jdbc:h2:mem:testdb`
- [ ] View AGENTS, ORDERS, REASSIGNMENT_SUGGESTIONS tables
- [ ] Verify seed data is present

## 🖥️ Frontend Setup

### Dependencies
```bash
cd frontend
npm install
```
- [ ] npm install completes successfully
- [ ] `node_modules/` directory created
- [ ] No peer dependency warnings (warnings OK, errors should be fixed)

### Dev Server
```bash
npm run dev
```
- [ ] Dev server starts
- [ ] Logs show "Local: http://localhost:5173"
- [ ] No VITE errors

### Open UI
```
http://localhost:5173
```
- [ ] Page loads without errors
- [ ] Header shows "AI Reassignment Engine"
- [ ] Tabs visible: Reassignments, Agent Roster
- [ ] Auto-refresh toggle visible
- [ ] Refresh button visible

### Verify Data Loads
- [ ] Go to "Agent Roster" tab
  - [ ] All 5 agents listed in table
  - [ ] Status column shows AVAILABLE/BUSY
  - [ ] Active Orders column has numbers
  - [ ] Status dropdown works

- [ ] Go to "Reassignments" tab
  - [ ] Shows message "No pending reassignment suggestions at this time"
  - [ ] This is correct (no suggestions yet)

## 🧪 Agentic Loop Test

### Test: Agent Goes OFFLINE

1. In frontend, go to **Agent Roster** tab
2. Find **AGT-001** (Priya Sharma) - status should be BUSY
3. Change dropdown from BUSY to **OFFLINE**
   - [ ] Dropdown changes immediately
   - [ ] Check backend logs: should see "Agent AGT-001 status updated: BUSY → OFFLINE"
   - [ ] Check backend logs: should see "Publishing AgentOfflineEvent for agent AGT-001"

4. Switch to **Reassignments** tab
5. Watch for 3 seconds (or click Refresh)
   - [ ] New suggestion appears
   - [ ] Suggestion shows ORD-001 and/or ORD-002 and/or ORD-008 (orders assigned to AGT-001)
   - [ ] Suggestion has **⚡ Re-plan** badge (orange, on right side)
   - [ ] Suggestion shows confidence score (green/amber/red bar)
   - [ ] "PENDING" badge visible

6. Click **▶ Details** to expand suggestion
   - [ ] AI Reasoning text appears
   - [ ] Recommendation shows an agent ID (AGT-002, AGT-003, AGT-004, or AGT-005)
   - [ ] Confidence percentage shows (e.g., 0.75, 0.90)
   - [ ] "Accept & Reassign" and "Reject" buttons visible

7. Click **Accept & Reassign**
   - [ ] Suggestion disappears from list
   - [ ] Reassignments tab shows "No pending reassignment suggestions"
   - [ ] Order is now reassigned (not visible in UI but backend reflects change)

### Success Criteria
- [ ] Event triggered when agent → OFFLINE
- [ ] Async re-plan ran (affected orders identified)
- [ ] Suggestions created with AGENT_OFFLINE trigger reason
- [ ] UI showed ⚡ badge (proving trigger reason detection works)
- [ ] Ops could approve/reject suggestion

## 🐛 If Something Breaks

### Backend won't start
- [ ] Check Java version: `java -version` (need 17+)
- [ ] Check Maven: `mvn -version`
- [ ] Try: `mvn clean` then `mvn install`
- [ ] Check port 8080 not in use: `lsof -i :8080`

### Frontend shows CORS errors in console
- [ ] Verify backend IS running on http://localhost:8080
- [ ] Check `frontend/src/services/api.js` has correct API_BASE
- [ ] Verify backend `application.properties` has CORS enabled for localhost:5173

### Suggestions not appearing after agent OFFLINE
- [ ] Check backend logs for "@EventListener" messages
- [ ] Check async executor logs ("replan-" thread names)
- [ ] Try manual refresh (button in top right)
- [ ] Toggle auto-refresh off/on
- [ ] Verify agent status change was successful (check roster)

### AI reasoning shows generic text instead of LLM response
- [ ] This usually means AI strategy failed and rule-based was used
- [ ] Check backend logs for "AI strategy failed, falling back"
- [ ] Verify LLM credentials are set correctly
- [ ] Verify network access to `https://uat-saksham.godrejcapital.com`
- [ ] This is OK for demo (fallback resilience is a feature!)

### Port 8080 or 5173 already in use
- [ ] Backend: Change `server.port` in `backend/src/main/resources/application.properties`
- [ ] Frontend: Run `npm run dev -- --port 5174`

## ✅ Final Verification

Once all checks pass:

1. **Backend running**: ✓
2. **Frontend running**: ✓
3. **Seed data loaded**: ✓
4. **Agentic loop working**: ✓
   - Agent offline → suggestions appear
   - ⚡ badge visible
   - AI reasoning visible
   - Accept/reject works

5. **Ready for demo**: ✓

---

## 📝 Demo Script

Use this script for the 5-minute demo:

```
0:00-0:15 | Setup
- Show backend running (logs visible)
- Show frontend dashboard
- Point to Agent Roster with all 5 agents

0:15-1:00 | Initial State
- Show all agents AVAILABLE/BUSY
- Show Reassignments tab (empty, "No pending suggestions")
- Explain seed data: 8 orders pre-assigned

1:00-2:00 | Trigger Agentic Loop
- Go to Agent Roster
- Find AGT-001, show status is BUSY with 2 active orders
- Change status dropdown to OFFLINE
- Switch to Reassignments tab
- NEW suggestions appear within 3 seconds
- Point to ⚡ Re-plan badge (automatic re-planning)

2:00-3:30 | Inspect Suggestion
- Expand one suggestion with "Details"
- Show:
  - Order ID (ORD-001 or ORD-002 or ORD-008)
  - Recommended agent (from rule-based or AI)
  - Confidence score (0-100% bar)
  - AI reasoning (full explanation text)
- Explain: This is AI's recommendation + confidence

3:30-4:30 | Approval Flow
- Click "Accept & Reassign"
- Suggestion disappears (status → ACCEPTED)
- Agent Roster shows AGT-001 still OFFLINE
- Explain: System queues suggestions, ops approves
  (human in the loop)

4:30-5:00 | Explain Architecture
- Agentic loop: event-driven, async, non-blocking
- Routing strategies: switchable at runtime
- AI resilience: falls back to rule-based if LLM fails
- Ready for Spring 2 zones & Spring 3 SLA features
```

---

**You're ready to test and demo!**
