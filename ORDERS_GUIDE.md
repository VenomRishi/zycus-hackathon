# How to Manage Orders - Complete Guide

## 📋 What You Can Do Now

### 1. **View All Orders** (Orders Tab)
- Frontend shows table with all orders
- Columns: Order ID, Description, Status, Assigned Agent, Agent Name, Created Date
- Filter by status: All, Assigned, Re-planning, Reassigned, Delivered

### 2. **See Which Agent is Servicing Which Order**
In the **Orders tab**, you can see:
- Order ID (ORD-001, ORD-002, etc.)
- Order Description (pickup location → dropoff location)
- **Assigned Agent ID** (e.g., AGT-001)
- **Agent Name** (e.g., Priya Sharma)
- **Agent Status** (AVAILABLE, BUSY, OFFLINE) - shown with colored badge
- **Agent Load** (e.g., "2 active orders")

### 3. **Create New Orders** (Via API)

**Using curl:**
```bash
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{
    "id": "ORD-099",
    "description": "Fresh Groceries - Koramangala to Whitefield",
    "assignedAgentId": "AGT-002"
  }'
```

**Replace with:**
- `id`: Unique order ID (ORD-101, ORD-102, etc.)
- `description`: What's being delivered
- `assignedAgentId`: Which agent gets the order (AGT-001 through AGT-005, or any available agent)

After creating, refresh the frontend (auto-refresh enabled or click Refresh button):
- New order appears in **Orders tab**
- Agent's active order count increases in **Agent Roster**

### 4. **Assign Orders to Available Agents**

**Best practice:**
1. Go to **Agent Roster** tab
2. Check which agents are **AVAILABLE** (green badge)
3. Check their **Active Orders** count (lower = more available)
4. Create new order and assign to that agent

**Example:**
```bash
# Agent AGT-002 (Rahul Verma) is AVAILABLE with 0 active orders
# Perfect candidate for a new order

curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{
    "id": "ORD-110",
    "description": "Electronics Package - MG Road to JP Nagar",
    "assignedAgentId": "AGT-002"
  }'
```

## 🔄 Watch the Full Workflow

### Scenario: Agent Gets Sick Mid-Shift

**Step 1: Create orders**
```bash
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"id": "ORD-201", "description": "Food Delivery A", "assignedAgentId": "AGT-001"}'

curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"id": "ORD-202", "description": "Food Delivery B", "assignedAgentId": "AGT-001"}'
```

**Step 2: Open UI**
- Go to http://localhost:5173
- Go to **Orders tab**
- See ORD-201, ORD-202 assigned to AGT-001 (Priya Sharma)

**Step 3: Agent Gets Sick**
- Go to **Agent Roster tab**
- Change AGT-001 status to **OFFLINE**

**Step 4: Watch Agentic Loop**
- Go to **Reassignments tab**
- Within 3 seconds, two new suggestions appear (for ORD-201 and ORD-202)
- Both have **⚡ Re-plan** badge (automatic, not manual)
- Each shows recommended agent + confidence + reasoning

**Step 5: Ops Approves**
- Click "Accept & Reassign" on each suggestion
- Go back to **Orders tab**
- Refresh: Orders now show new assigned agents (AGT-002, AGT-003, etc.)

---

## 📊 Order Status Lifecycle

```
ASSIGNED
↓
[Agent is actively delivering]
↓
Remaining in ASSIGNED until reassignment needed
↓
(Agent goes offline or manual reassignment request)
↓
REASSIGNMENT_PENDING
↓
[AI suggestion + ops approval]
↓
(Ops accepts suggestion)
↓
REASSIGNED
↓
[Agent (now different one) delivers]
↓
DELIVERED
```

---

## 🎯 Key Things to See

### 1. **Orders Tab** - See All Assignments
| Order | Description | Status | Assigned Agent | Agent Name | Agent Status |
|-------|-------------|--------|-----------------|------------|--------------|
| ORD-001 | Electronics... | ASSIGNED | AGT-001 | Priya Sharma | BUSY |
| ORD-002 | Groceries... | ASSIGNED | AGT-001 | Priya Sharma | BUSY |
| ORD-003 | Pharma... | ASSIGNED | AGT-003 | Ananya Iyer | BUSY |
| ORD-004 | Documents... | REASSIGNMENT_PENDING | AGT-005 | Deepak Mehta | OFFLINE |

### 2. **Agent Roster** - See Capacity
- **AGT-001**: BUSY, 2 active orders
- **AGT-002**: AVAILABLE, 0 active orders ← Good candidate for new order
- **AGT-003**: BUSY, 1 active order
- **AGT-004**: AVAILABLE, 0 active orders ← Good candidate for new order
- **AGT-005**: OFFLINE, 3 active orders ← Orders need reassignment

### 3. **Reassignments** - See AI Recommendations
- Shows pending suggestions (with ⚡ badge for automatic ones)
- Shows confidence score + AI reasoning
- Ops approves/rejects

---

## 💡 Tips

### To See Orders at Each Stage
```bash
# All ASSIGNED orders
curl http://localhost:8080/orders?status=ASSIGNED

# Orders pending reassignment
curl http://localhost:8080/orders?status=REASSIGNMENT_PENDING

# Already reassigned orders
curl http://localhost:8080/orders?status=REASSIGNED

# Delivered
curl http://localhost:8080/orders?status=DELIVERED
```

### To Get Order Details
```bash
curl http://localhost:8080/orders/ORD-001
```

### To See Which Orders are Affected When Agent Goes Offline
1. Go to **Agent Roster**
2. Note which agent is BUSY with how many orders
3. Change that agent to OFFLINE
4. Go to **Orders tab**, filter by REASSIGNMENT_PENDING
5. You'll see all orders that were affected
6. Go to **Reassignments tab**
7. You'll see suggestions for those exact orders

---

## 🚀 Complete Test Flow

```
1. Open frontend: http://localhost:5173

2. Go to "Orders" tab
   ✓ See 8 seed orders assigned to agents

3. Go to "Agent Roster" tab
   ✓ See 5 agents with their status and active orders

4. Create new order via curl:
   curl -X POST http://localhost:8080/orders \
   -H "Content-Type: application/json" \
   -d '{"id":"ORD-999", "description":"New Order", "assignedAgentId":"AGT-002"}'

5. Refresh Orders tab
   ✓ New order appears (9 total now)
   ✓ AGT-002's active order count increases

6. Go to Agent Roster
   ✓ AGT-002 now shows higher active order count

7. Change AGT-002 status to OFFLINE
   ✓ Watch Reassignments tab
   ✓ See suggestion appear with ⚡ badge

8. Accept suggestion
   ✓ Go back to Orders tab
   ✓ ORD-999 now shows different agent assigned

9. Done! You've seen the full workflow:
   Create → Assign → Agent Offline → AI Suggests → Approve → Reassigned
```

---

## 📝 Seed Data (Already Loaded)

**5 Agents:**
- AGT-001: Priya Sharma (BUSY, 2 orders)
- AGT-002: Rahul Verma (AVAILABLE, 0 orders)
- AGT-003: Ananya Iyer (BUSY, 1 order)
- AGT-004: Kiran Nair (AVAILABLE, 0 orders)
- AGT-005: Deepak Mehta (BUSY, 3 orders)

**8 Orders:**
- ORD-001, ORD-002, ORD-008 → AGT-001
- ORD-003, ORD-007 → AGT-003
- ORD-004, ORD-005, ORD-006 → AGT-005

---

Now you can:
✅ See all orders and their assigned agents
✅ Create new orders and assign to available agents
✅ Watch agents and their order load
✅ Trigger reassignment by making agent OFFLINE
✅ See AI recommendations in Reassignments tab
✅ Approve/reject reassignments
