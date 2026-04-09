# Comic-Con Parking Management System - ER-Diagram

This repository contains the Entity-Relationship (ER) diagram for a multi-zone parking management system designed for Comic-Con India. The system is built to handle thousands of visitors arriving across multiple days using various vehicle types including bikes, cars, SUVs, cabs, and EVs.

The diagram models how vehicles are tracked from entry to exit, how parking spots are allocated based on vehicle type and access category, how parking sessions and tickets are generated, and how payments are recorded and calculated.

## Entities

### 1. `vehicle_categories`
**Purpose:** Defines the different types of vehicles that are allowed to enter the parking facility.

**Why it exists:** The parking system needs to know what kind of vehicle is entering so it can assign a compatible parking spot. A bike cannot be sent to an SUV-sized spot, and an EV needs access to a charging-enabled spot. Rather than storing vehicle type as a plain text string inside the vehicles table, it is separated into its own table so that types are consistent and reusable.

**Attributes:**
| Attribute | Type | Description |
|---|---|---|
| `id` | string (PK) | Unique identifier for each vehicle category |
| `name` | string | Name of the category e.g. Bike, Car, SUV, Cab, EV |
| `description` | string | Short description of the vehicle type |
| `rate_per_hour` | decimal | Hourly parking charge for this vehicle type, used to calculate the fee when a session ends |

---

### 2. `vehicles`
**Purpose:** Stores information about every individual vehicle that enters the parking facility.

**Why it exists:** The system must track which vehicles entered, when, and how many times. A vehicle can visit the venue on multiple days across the event. Keeping a vehicle record means the system does not create duplicate entries every time the same car enters — it reuses the existing vehicle record and creates a new parking session instead.

**Attributes:**
| Attribute | Type | Description |
|---|---|---|
| `id` | string (PK) | Unique identifier for each vehicle |
| `license_plate` | string | The vehicle's registration number, used to identify it at entry |
| `vehicle_category_id` | string (FK) | Links to `vehicle_categories` to identify what type of vehicle this is |
| `owner_name` | string | Name of the vehicle owner or driver |
| `contact_number` | string | Contact number for the owner, useful for notifications or disputes |

---

### 3. `parking_zones`
**Purpose:** Represents the top-level physical divisions of the parking facility at the venue.

**Why it exists:** A large convention venue like Comic-Con India has a massive parking area that is split into multiple named zones (for example Zone A, Zone B, Zone C). Zones help staff direct vehicles to the right area of the venue and help the system track availability at a high level.

**Attributes:**
| Attribute | Type | Description |
|---|---|---|
| `id` | string (PK) | Unique identifier for each zone |
| `name` | string | Name or label of the zone e.g. Zone A, North Wing |
| `description` | string | Additional details about the zone such as its location or purpose |

---

### 4. `parking_levels`
**Purpose:** Represents individual floors or levels within each parking zone.

**Why it exists:** Within each zone, the parking facility may have multiple floors or levels such as Ground Floor, Level 1, Level 2. A level always belongs to one zone. Actual parking spots are assigned at the level, not at the zone, so this entity acts as a bridge between zones and spots. This makes the location hierarchy clear: Zone → Level → Spot.

**Attributes:**
| Attribute | Type | Description |
|---|---|---|
| `id` | string (PK) | Unique identifier for each level |
| `zone_id` | string (FK) | Links to `parking_zones` to indicate which zone this level belongs to |
| `level_number` | int | Numeric floor number e.g. 0 for ground, 1 for first floor |
| `name` | string | Human-readable name for the level e.g. Ground Floor, Level 1 |

---

### 5. `spot_categories`
**Purpose:** Defines the different categories of parking spots, including which ones are reserved for special access groups.

**Why it exists:** Comic-Con India has multiple special groups who need dedicated parking — cosplayers carrying large props, exhibitors with heavy equipment, VIP guests, staff members, and EV vehicle owners who need charging points. A plain parking spot cannot represent this. Spot categories allow the system to tag each spot with its access type, enforce reservations, and ensure that a VIP spot is never accidentally assigned to a regular visitor.

**Attributes:**
| Attribute | Type | Description |
|---|---|---|
| `id` | string (PK) | Unique identifier for each spot category |
| `name` | string | Name of the category e.g. General, VIP, Exhibitor, Staff, Cosplayer, EV |
| `description` | string | Details about who this category is meant for |
| `is_reserved` | boolean | True if the spot is reserved for a specific group, False for general public |
| `access_type` | string | Specifies the exact access group: PUBLIC, VIP, EXHIBITOR, COSPLAYER, STAFF, EV |

---

### 6. `parking_spots`
**Purpose:** Represents every individual physical parking spot in the entire facility.

**Why it exists:** This is the core physical resource of the system. Each parking spot has a fixed location (on a specific level in a specific zone), is designed for a specific vehicle type, and belongs to a specific access category. The system uses this entity to check availability, assign spots to incoming vehicles, and release spots when vehicles exit. Without this entity, there is no way to know which spot is free, which is occupied, or which is reserved.

**Attributes:**
| Attribute | Type | Description |
|---|---|---|
| `id` | string (PK) | Unique identifier for each parking spot |
| `level_id` | string (FK) | Links to `parking_levels` to show which level this spot is on |
| `spot_category_id` | string (FK) | Links to `spot_categories` to define access type and reservation status |
| `vehicle_category_id` | string (FK) | Links to `vehicle_categories` to define which vehicle type fits this spot |
| `spot_number` | string | The physical label on the spot e.g. A-101, B-204 |
| `is_available` | boolean | True if the spot is currently free, False if occupied |
| `has_ev_charging` | boolean | True if the spot has an EV charging point installed |

---

### 7. `parking_sessions`
**Purpose:** Records every single visit a vehicle makes to the parking facility, from entry to exit.

**Why it exists:** This is the most important transactional entity in the system. Every time a vehicle enters, a new session is created. Every time it exits, the session is closed with an exit timestamp and a calculated fee. A vehicle can have many sessions across multiple days of the event. A parking spot can also be involved in many sessions over time as different vehicles use it. This entity answers the most critical operational questions: which vehicles are currently inside, when did they enter, when did they leave, and how much should they be charged.

**Attributes:**
| Attribute | Type | Description |
|---|---|---|
| `id` | string (PK) | Unique identifier for each parking session |
| `vehicle_id` | string (FK) | Links to `vehicles` to identify which vehicle this session is for |
| `parking_spot_id` | string (FK) | Links to `parking_spots` to identify the assigned spot |
| `entry_time` | timestamp | Date and time when the vehicle entered the facility |
| `exit_time` | timestamp | Date and time when the vehicle exited, null while session is active |
| `is_active` | boolean | True if the vehicle is still parked inside, False after exit |
| `calculated_fee` | decimal | The total parking fee computed at exit based on duration and rate |

---

### 8. `parking_tickets`
**Purpose:** Represents the ticket issued to a vehicle when it enters the parking facility.

**Why it exists:** The assignment specifies that tickets and parking sessions should be separate structures. In a real event parking system, a physical or digital ticket is generated at entry and carries a ticket number that the driver holds onto. This ticket is linked to the session but is its own record. Keeping tickets separate from sessions also allows for edge cases like a lost ticket being reissued, which would create a second ticket record for the same session without corrupting the session data.

**Attributes:**
| Attribute | Type | Description |
|---|---|---|
| `id` | string (PK) | Unique identifier for each ticket |
| `session_id` | string (FK) | Links to `parking_sessions` to associate the ticket with a session |
| `ticket_number` | string | The human-readable ticket code given to the driver e.g. TKT-00123 |
| `issued_at` | timestamp | Date and time when the ticket was generated |

---

### 9. `payments`
**Purpose:** Records the payment made by a vehicle owner at the end of a parking session.

**Why it exists:** The system needs to track whether a session has been paid for, how much was paid, through what method, and when. Payment is separate from the session itself because the session captures time-based data (entry, exit, duration) while the payment captures financial data (amount, method, status). Keeping them separate also supports scenarios like failed payments being retried, which would create a second payment record for the same session rather than overwriting the original attempt.

**Attributes:**
| Attribute | Type | Description |
|---|---|---|
| `id` | string (PK) | Unique identifier for each payment record |
| `session_id` | string (FK) | Links to `parking_sessions` to associate the payment with a session |
| `amount` | decimal | The amount charged for the parking session |
| `payment_method` | string | How the payment was made e.g. Cash, UPI, Card, Wallet |
| `payment_status` | string | Current status of the payment e.g. Pending, Completed, Failed |
| `paid_at` | timestamp | Date and time when the payment was successfully completed |

---

## Relationships

| Relationship | Type | Meaning |
|---|---|---|
| `vehicles` → `vehicle_categories` | Many-to-One | Many vehicles can belong to one category |
| `parking_levels` → `parking_zones` | Many-to-One | Many levels can exist within one zone |
| `parking_spots` → `parking_levels` | Many-to-One | Many spots can exist on one level |
| `parking_spots` → `spot_categories` | Many-to-One | Many spots can share one category |
| `parking_spots` → `vehicle_categories` | Many-to-One | Many spots can be designated for one vehicle type |
| `parking_sessions` → `vehicles` | Many-to-One | One vehicle can have many sessions across multiple days |
| `parking_sessions` → `parking_spots` | Many-to-One | One spot can be used across many sessions over time |
| `parking_tickets` → `parking_sessions` | Many-to-One | A session can have more than one ticket e.g. if reissued |
| `payments` → `parking_sessions` | Many-to-One | A session can have more than one payment record e.g. if retried |

---

                                                Thank You 
