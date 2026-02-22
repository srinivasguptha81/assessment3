# 🍕 Module 2 — Smart Food Stall Pre-Ordering System

![Python](https://img.shields.io/badge/Python-3.11-blue?style=flat-square&logo=python)
![Django](https://img.shields.io/badge/Django-5.x-green?style=flat-square&logo=django)
![SQLite](https://img.shields.io/badge/Database-SQLite-lightgrey?style=flat-square)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=flat-square)

> Part of the **LPU Smart Campus Management System** — a multi-module Django project built for university digitization.

---

## 📌 Problem Statement

LPU has multiple food stalls across campus. During lunch and morning breaks, students crowd stalls simultaneously, causing long queues and wasted break time. Stall owners have no way to predict demand or prepare in advance.

**This module solves it** by letting students pre-order food and choose a pickup time slot — so stalls can prepare ahead, queues disappear, and students collect food at their break without waiting.

---

## ✨ Features

### 🎓 Student Side
- Browse all open campus food stalls with photos, location, and hours
- View full menu with item photos, veg/non-veg indicators, and prep time
- Add items to a session cart (no login required per item — cart is session-based)
- Choose a pre-defined break time slot for pickup
- Add special instructions (e.g. "no onions")
- Live slot availability — shows how many slots remain, warns when nearly full
- Real-time order tracking with a 4-step progress stepper (Placed → Confirmed → Ready → Collected)
- Auto-refresh every 30 seconds while order is active
- Full order history with status badges

### 🏪 Stall Owner Side
- **Manage Menu** — add, edit, show/hide, and delete menu items from a clean UI (no Django admin needed)
- Organize items into categories with custom emoji icons
- Upload item photos directly from the interface
- **Orders Dashboard** — view all today's orders grouped by pickup slot
- Update order status live with a dropdown (Pending → Confirmed → Ready → Collected)
- AJAX status updates — no page reload needed
- Dashboard auto-refreshes every 45 seconds
- Revenue summary for the day

### 🤖 AI Demand Prediction
- Tracks hourly order volume for each stall automatically
- **7-day moving average algorithm** predicts expected orders for the next hour
- Confidence scoring based on variance (High / Medium / Low)
- Visual bar chart showing peak hours across the last 7 days
- Actionable advice: "Prepare approximately X orders for 13:00"

---

## 🗂️ Project Structure

```
lpu_cms/
├── food/
│   ├── models.py           # 7 database models
│   ├── views.py            # 18 views — ordering, owner, AJAX, AI
│   ├── urls.py             # 15 URL routes
│   ├── forms.py            # OrderForm, OrderStatusForm
│   └── admin.py            # Admin with inline OrderItems
│
├── templates/food/
│   ├── stall_list.html         # All stalls on campus
│   ├── stall_menu.html         # Menu with photos + add-to-cart
│   ├── cart.html               # Cart review + slot selection
│   ├── order_detail.html       # Live order tracking
│   ├── my_orders.html          # Student order history
│   ├── owner_dashboard.html    # Stall owner orders view
│   ├── demand_analytics.html   # AI demand chart
│   └── manage_menu.html        # Menu management UI
│
└── attendance/
    └── context_processors.py   # Injects is_stall_owner, faculty, student into all templates
```

---

## 🗄️ Database Models

| Model | Purpose |
|-------|---------|
| `FoodStall` | Each physical stall — name, location, owner, hours |
| `Category` | Groups menu items (e.g. Snacks 🍟, Drinks ☕) |
| `MenuItem` | Individual food item — price, photo, veg/non-veg, prep time |
| `BreakSlot` | Pre-defined pickup slots with max capacity per slot |
| `Order` | One student's complete pre-order for a specific slot |
| `OrderItem` | Line item within an order (quantity × menu item) |
| `DemandRecord` | Hourly order count per stall — feeds AI prediction |

---

## 🔗 URL Routes

| URL | View | Description |
|-----|------|-------------|
| `/food/` | `stall_list` | All open stalls |
| `/food/stall/<id>/` | `stall_menu` | Menu for one stall |
| `/food/cart/<stall_id>/` | `cart_view` | Cart review + slot picker |
| `/food/checkout/<stall_id>/` | `checkout` | Place the order |
| `/food/orders/` | `my_orders` | Student order history |
| `/food/orders/<id>/` | `order_detail` | Live order tracking |
| `/food/owner/` | `owner_dashboard` | Stall owner — today's orders |
| `/food/owner/analytics/` | `demand_analytics` | AI demand chart |
| `/food/menu/` | `manage_menu` | Menu management (owner) |
| `/food/menu/add/` | `add_item` | Add menu item |
| `/food/menu/edit/<id>/` | `edit_item` | Edit menu item |
| `/food/menu/toggle/<id>/` | `toggle_item` | Show/hide item |
| `/food/menu/delete/<id>/` | `delete_item` | Delete item |
| `/food/menu/category/add/` | `add_category` | Add category |
| `/food/api/cart/add/<id>/` | `add_to_cart` | AJAX — add to cart |
| `/food/api/cart/remove/<id>/` | `remove_from_cart` | AJAX — remove from cart |
| `/food/api/order/<id>/status/` | `update_order_status` | AJAX — update status |
| `/food/api/slots/<stall_id>/` | `slot_availability` | AJAX — slot counts |

---

## 🤖 AI Algorithm — Moving Average Demand Prediction

The AI demand predictor uses a **simple moving average** over a 7-day window.

### How It Works

```
For the next hour H on today's date:
  1. Collect order counts for hour H on each of the past 7 days
  2. predicted = sum(counts) / 7
  3. variance  = sum((x - mean)² for x in counts) / 7
  4. confidence = High if variance < 5, Medium if < 20, else Low
```

### Example

| Day | Orders at 13:00 |
|-----|----------------|
| Mon | 18 |
| Tue | 22 |
| Wed | 19 |
| Thu | 24 |
| Fri | 20 |
| Sat | 12 |
| Sun | 15 |

**Predicted for next Monday 13:00 = (18+22+19+24+20+12+15) / 7 ≈ 19 orders**

### Why Moving Average?
- Simple and interpretable — stall owners can understand it without a data science background
- Adapts over time — as more data is collected, predictions improve
- No external ML library needed — pure Python math
- This is the foundation of time-series forecasting used in production systems like Google Trends

### Viva Answer
> *"The AI uses a 7-day moving average time-series model. It collects order counts for the same hour across the past 7 days and averages them to predict next hour demand. Confidence is determined by variance — low variance means consistent demand, high variance means unpredictable. This is stored in the `DemandRecord` model which is updated on every successful checkout."*

---

## 🛒 Cart System

The cart uses **Django sessions** — no database writes until checkout.

```python
# Cart structure in session:
request.session['cart'] = { 'item_id': quantity }
# Example:
{ '3': 2, '7': 1 }   # 2x item 3, 1x item 7
```

**Why sessions (not a DB model)?**
- Faster — no DB query for every add/remove
- Auto-expires — cart clears when session ends
- Simpler — no "abandoned cart" cleanup needed

---

## 👥 User Roles

| Role | Access |
|------|--------|
| **Student** | Browse stalls, add to cart, checkout, track orders, view history |
| **Stall Owner** | Manage menu, view today's orders, update status, see AI analytics |
| **Admin** | Everything via Django admin + all above |

Role detection uses a **context processor** (`attendance/context_processors.py`) that injects `is_stall_owner`, `faculty`, and `student` variables into every template automatically.

---

## ⚙️ Setup & Run

```bash
# 1. Activate virtual environment
lpu_env\Scripts\activate          # Windows
source lpu_env/bin/activate       # Mac/Linux

# 2. Install dependencies
pip install -r requirements.txt

# 3. Run migrations
python manage.py migrate

# 4. Create superuser
python manage.py createsuperuser

# 5. Start server
python manage.py runserver
```

---

## 🧪 Test Data Setup (Admin Panel)

1. `/admin/food/foodstall/` → Add stall → set **Owner** to `vendor1` user
2. `/admin/food/breakslot/` → Add slots e.g. *Morning Break 10:30–11:00 (max 30)*
3. `/food/menu/` (logged in as vendor1) → Add categories and menu items with photos
4. Log in as a student → browse → order → track
5. Log in as vendor1 → `/food/owner/` → update order status

---

## 🔑 Key Django Concepts Used

| Concept | Where Used |
|---------|-----------|
| `Session-based cart` | `get_cart()` / `save_cart()` — no DB needed |
| `JsonResponse` | AJAX cart add/remove, order status update |
| `json.loads(request.body)` | Reading JSON from AJAX POST requests |
| `update_or_create` | Safe upsert on DemandRecord per hour |
| `defaultdict` | Grouping orders by slot in owner dashboard |
| `@property` | `is_full`, `slots_left`, `subtotal` — computed fields |
| `aggregate(Sum(...))` | Total revenue calculation |
| `auto_now_add / auto_now` | Auto timestamps on Order create/update |
| `unique_together` | One demand record per stall per hour per date |
| `related_name` | `order.items.all()`, `stall.menu_items.all()` |
| Context processor | Injects role variables into every template |

---

## 📁 Media Files

Uploaded item and stall photos are stored in `media/`:

```
media/
├── menu_photos/     ← MenuItem.photo uploads
├── stall_photos/    ← FoodStall.photo uploads
```

Served during development via `urls.py`:
```python
urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

## 📄 Related Modules

| Module | Description |
|--------|-------------|
| [Module 1](../attendance/) | Smart Attendance System with AI Face Recognition |
| Module 2 | **Smart Food Stall Pre-Ordering System** ← we are here |
| Module 3 | Smart Resource & Room Booking System *(coming soon)* |
