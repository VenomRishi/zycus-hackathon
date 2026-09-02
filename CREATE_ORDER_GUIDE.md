# Create Orders with AI-Suggested Agents

A complete guide to creating new orders with AI-powered agent recommendations.

## 🎯 What You Can Do Now

### **Create New Order with AI Agent Recommendation**
1. Click **"+ New Order"** button (top right)
2. Fill in order details (description, pickup zone, dropoff zone)
3. Click **"Get Agent Suggestions"**
4. AI shows ranked list of available agents with:
   - Confidence score (0-100%)
   - Why this agent (AI reasoning)
   - Current load (active orders)
   - Agent status (AVAILABLE/BUSY)
5. Select best agent
6. Click **"Create & Assign"**
7. Order is created and assigned to that agent

---

## 📝 Step-by-Step Walkthrough

### **Step 1: Open Create Order Modal**
- Click the **"+ New Order"** button (top right of dashboard)
- Modal opens with form

### **Step 2: Fill Order Details**

**Order ID** (optional)
- Auto-generated if left blank (e.g., ORD-4523)
- Or enter custom ID (must be unique)

**Order Description** ⭐ Required
- What's being delivered
- Example: "Fresh Groceries - Downtown Mart to Airport Terminal"
- Example: "Electronics Package - Tech Hub to Office Complex"

**Pickup Zone** (optional)
- Where order is picked from
- Example: "Downtown", "North District", "Central"

**Dropoff Zone** (optional)
- Where order is delivered to
- Example: "Airport", "Suburbs", "Industrial Area"

**Example Input:**
```
Order ID: (leave blank - auto-generate)
Description: Fresh Groceries - Mart to Residence
Pickup Zone: Downtown
Dropoff Zone: Residential Area
```

### **Step 3: Get AI Suggestions**
- Click **"Get Agent Suggestions"**
- System calls backend to suggest available agents
- Shows loading spinner while AI analyzes

### **Step 4: Review Agent Recommendations**

**For each agent, you see:**

| Field | What It Shows |
|-------|---------------|
| **Rank** | #1 = Best, #2 = Second best, etc. |
| **Agent Name** | Full name (e.g., Priya Sharma) |
| **Agent ID** | Unique ID (e.g., AGT-001) |
| **Status Badge** | AVAILABLE (green), BUSY (amber), OFFLINE (red) |
| **Active Orders** | How many orders they're currently carrying |
| **Confidence Score** | 0-100% confidence (AI's assessment) |
| **Reasoning** | Why AI recommends this agent (plain English) |

**Example Recommendation:**
```
#1 - Rahul Verma (AGT-002)
Status: AVAILABLE
Active Orders: 0
Confidence: 95%
Reasoning: Agent is available with no current orders, 
           making this an optimal assignment for 
           immediate pickup and delivery.
```

### **Step 5: Select Best Agent**
- Click on any suggestion to select it
- Selected agent shows "✓ Selected" badge
- Top suggestion is pre-selected by default

### **Step 6: Create & Assign**
- Click **"Create & Assign"** button
- Loading spinner appears
- Order is created in backend
- Order is assigned to selected agent
- Modal closes automatically

### **Step 7: See New Order**
- UI automatically navigates to **Orders** tab
- Refresh page or wait for auto-refresh
- New order appears in table with:
  - Order ID (e.g., ORD-4523)
  - Description you entered
  - Status: **ASSIGNED**
  - Assigned Agent: The one you selected (e.g., AGT-002)
  - Agent Name: Rahul Verma
  - Agent Status: AVAILABLE (shows at creation time)

---

## 💡 Real-World Example Scenario

### **Scenario: New Grocery Order Arrives**

**You:**
1. Click "+ New Order"
2. Fill form:
   - Description: "Groceries - Farm Market to Downtown Office"
   - Pickup Zone: "Farmer Market District"
   - Dropoff Zone: "Downtown"
3. Click "Get Agent Suggestions"

**System suggests:**
```
#1 - Kiran Nair (AGT-004) - Confidence: 92%
   Status: AVAILABLE, 0 active orders
   "Agent Kiran is currently available in the Downtown 
    zone with no active orders, ideal for rapid pickup 
    and delivery."

#2 - Rahul Verma (AGT-002) - Confidence: 85%
   Status: AVAILABLE, 0 active orders
   "Agent Rahul is available but currently in North zone, 
    would require zone transition but remains viable."

#3 - Priya Sharma (AGT-001) - Confidence: 62%
   Status: BUSY, 2 active orders
   "Agent Priya is currently busy with 2 orders but can 
    be considered if immediate availability is not critical."
```

**You pick:**
- Click on Kiran Nair (top suggestion, 92% confidence)
- Click "Create & Assign"

**Result:**
- Order created: ORD-7421
- Assigned to: AGT-004 (Kiran Nair)
- Status: ASSIGNED
- Kiran now shows 1 active order in Agent Roster

---

## 🔄 Watch It Work End-to-End

### **Complete Workflow**

```
1. Click "+ New Order"
   ↓
2. Enter order details
   ↓
3. Click "Get Agent Suggestions"
   ↓
4. [Backend] Analyzes available agents
   ↓
5. [AI] Generates recommendations with confidence scores
   ↓
6. [UI] Shows ranked agent list
   ↓
7. Click on preferred agent
   ↓
8. Click "Create & Assign"
   ↓
9. [Backend] Creates order + assigns to selected agent
   ↓
10. [UI] Shows new order in Orders tab
    ↓
11. Go to Agent Roster → see agent's order count increased
    ↓
12. If agent goes OFFLINE → re-plan suggestions auto-appear
```

---

## 📊 What Happens After Order is Created

### **In Orders Tab**
- New order appears with:
  - Order ID
  - Description
  - Status: ASSIGNED
  - Assigned Agent: The one you selected
  - Agent Load: Increases by 1

### **In Agent Roster Tab**
- Selected agent's **Active Orders** count increases
- If agent was AVAILABLE, may change to BUSY (if now 2+ orders)

### **If Agent Goes Offline**
- Go to Agent Roster
- Change that agent to OFFLINE
- Go to Reassignments tab
- See new suggestion with **⚡ Re-plan** badge
- Accept suggestion to reassign order

---

## 🎓 How AI Makes Recommendations

The system evaluates agents on:

1. **Availability** - Agent status (AVAILABLE best, BUSY okay, OFFLINE worst)
2. **Current Load** - How many active orders they're carrying (fewer is better)
3. **Zone Match** - Is agent currently in pickup zone? (speeds delivery)
4. **Capacity** - How many orders can they take? (max capacity check)

**Confidence Score:**
- **90-100%** - Strong recommendation (available, low load, good zone fit)
- **70-89%** - Good recommendation (available but may have slight delays)
- **50-69%** - Acceptable (might be busy or far from pickup zone)
- **Below 50%** - Consider other options (may be offline or overloaded)

---

## 🚀 Test It Now

### **Quick Test Flow**

```bash
1. Open http://localhost:5173
2. Click "+ New Order"
3. Fill form:
   Order Description: "Coffee Delivery - Cafe to Office"
   Pickup Zone: "Downtown"
   Dropoff Zone: "Business District"
4. Click "Get Agent Suggestions"
   ↓ Wait 2-3 seconds for AI analysis
5. See suggestions appear ranked by confidence
6. Click on top suggestion (Rahul Verma, AGT-002 typically highest)
7. Click "Create & Assign"
8. Watch modal close and Orders tab auto-load
9. New order visible in table!
10. Go to Agent Roster → see that agent's active orders increased
```

---

## 💾 Backend API Endpoints

### **Get Agent Suggestions for New Order**
```
POST /orders/suggest-agent
Content-Type: application/json

{
  "description": "Order description here",
  "pickupZone": "Downtown",
  "dropoffZone": "Airport"
}

Response:
[
  {
    "agentId": "AGT-002",
    "agentName": "Rahul Verma",
    "status": "AVAILABLE",
    "activeOrderCount": 0,
    "confidence": 0.95,
    "reasoning": "Agent is available with no current orders...",
    "currentZone": "Downtown"
  },
  ...
]
```

### **Create Order**
```
POST /orders
Content-Type: application/json

{
  "id": "ORD-7421",
  "description": "Order description",
  "assignedAgentId": "AGT-002"
}

Response:
{
  "id": "ORD-7421",
  "description": "Order description",
  "assignedAgentId": "AGT-002",
  "status": "ASSIGNED",
  "createdAt": "2025-09-02T12:34:56"
}
```

---

## ⚠️ Common Questions

**Q: What if no agents are available?**
- Error shown: "No available agents found"
- Try again later when agents become available

**Q: Can I manually enter an agent ID?**
- No, you select from AI suggestions only
- This ensures AI validated the agent is actually available

**Q: What if I don't fill pickup/dropoff zones?**
- OK - system marks as "Unknown"
- AI still makes recommendations based on agent load
- Zones are optional (for Sprint 2 zone-aware routing)

**Q: Can I edit order after creating?**
- Currently no UI for editing
- You can reassign it (via re-planning)
- Or create a new order

**Q: How does AI suggest agents?**
- Uses active routing strategy (rule-based or AI)
- Rule-based: picks agent with fewest orders
- AI: calls LLM with agent context, gets confidence score

---

## 📈 Next Steps

After creating an order:

1. **Watch it in Orders tab** - See order with assigned agent
2. **Check Agent Roster** - See agent's load increased
3. **Simulate Agent Offline** - Change that agent to OFFLINE
4. **See Re-plan Suggestions** - Agentic loop auto-suggests new agent
5. **Approve Reassignment** - Order reassigned to different agent
6. **Verify in Orders tab** - See new agent assigned

---

## 🎉 You Now Have

✅ **AI-Powered Order Creation**
- Describe what needs delivery
- AI suggests best available agent
- See confidence score + reasoning
- Assign with one click

✅ **Full Order Lifecycle**
- Create → Assign → Track
- Agent goes offline? Auto-reassign
- All in one intuitive dashboard

Ready to test! Click "+ New Order" 🚀
