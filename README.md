# Frontistirio — Store Management Brain

> **Purpose:** Documents how stores (tutoring centers), groups, teaching periods, and store settings are structured and managed. This is the top-level organizational layer of the platform.

---

## Overview

The platform is multi-tenant. The hierarchy is:

```
Group (franchise/chain)
  └── Store (individual tutoring center)
        ├── TeachingPeriod (academic year/semester)
        ├── Students, Teachers, Courses, Classes
        └── StoreSettings (configuration)
```

A **SuperAdmin** manages Groups and Stores. A **store-user** manages everything within their Store.

---

## Models

### Store (`models/stores.js`)

```
Store {
  name              String
  group             → Group
  address           → Address
  user_id           → User (the store-user admin)
  teaching_periods  → [TeachingPeriod]
  isDeleted         Boolean
  deletedAt         Date
  createdBy / updatedBy → User
}
```

### Group (`models/group.js`)

```
Group {
  name      String
  stores    → [Store]
}
```

### TeachingPeriod (`models/teaching_periods.js`)

```
TeachingPeriod {
  name        String
  date_from   Date (required)
  date_to     Date (required)
  store_id    → Store
  default     Boolean     // only one period per store is "default" (active)
  isDeleted   Boolean
  deletedAt   Date
}
```

### StoreSettings (`models/store-settings.js`)

```
StoreSettings {
  store_id    → Store
  // (configuration fields — notification prefs, display options, etc.)
}
```

### Address (`models/address.js`)

```
Address {
  address   String
  number    Number
  zip_code  Number
  country   String
}
```

---

## API Endpoints

### Stores (`routes/stores.js`)

| Method | Path | Description | Auth |
|---|---|---|---|
| POST | `/stores/` | Create a store | superadmin |
| PUT | `/stores/:id` | Edit a store | superadmin |
| DELETE | `/stores/:id` | Delete a store | superadmin |
| GET | `/stores/` | Get all stores | superadmin |
| GET | `/stores/:id` | Get single store | superadmin, store-user |

### Groups (`routes/group.js`)

| Method | Path | Description | Auth |
|---|---|---|---|
| POST | `/group/` | Create a group | superadmin |
| PUT | `/group/:id` | Edit a group | superadmin |
| GET | `/group/` | Get all groups | superadmin |

### Teaching Periods (`routes/teaching_periods.js`)

| Method | Path | Description | Auth |
|---|---|---|---|
| POST | `/teaching-periods/` | Create a period | store-user |
| PUT | `/teaching-periods/:id` | Edit a period | store-user |
| DELETE | `/teaching-periods/:id` | Delete a period | store-user |
| GET | `/teaching-periods/` | Get store periods | store-user |
| PUT | `/teaching-periods/:id/set-default` | Set as active period | store-user |
| POST | `/teaching-periods/:id/copy-forward` | Copy data to new period | store-user |

### Store Settings (`routes/store-settings.js`)

| Method | Path | Description |
|---|---|---|
| GET | `/store-settings/` | Get store settings |
| PUT | `/store-settings/` | Update store settings |

---

## Store Creation Flow (SuperAdmin)

1. Create `Address` document
2. Create `Store` document (linked to group + address)
3. Update `Group.stores` to include the new store
4. All done in a **MongoDB transaction**

```javascript
const new_address = await addressHandler.createAddress(..., session);
const new_store = await storeHandler.createStore(..., session);
await groupsHandler.updateGroupStores(group_id, new_store._id, session);
await session.commitTransaction();
```

---

## Teaching Period System

Teaching periods are the core time unit. **Everything in the system is scoped to a period.**

### Default Period
- Each store has exactly **one default (active) period** at a time
- `TeachingPeriod.default = true`
- All API endpoints call `getStoreDefaultPeriod(store_id)` to get the active period
- Changing the active period changes what data the entire frontend sees

### Creating a New Period
1. `POST /teaching-periods/` with `date_from`, `date_to`
2. Period created and added to `Store.teaching_periods`
3. Done in a transaction

### Copying Forward to New Period
When starting a new school year:
1. `POST /teaching-periods/:id/copy-forward`
2. Copies students, courses, classes, teachers into the new period
3. Resets enrollments (you can choose what to carry over)

---

## Teaching Period — The Central Scoping Key

Almost every query in the system uses the default period:
```javascript
const default_period = await teachinPeriodHandler.getStoreDefaultPeriod(req.user?.store);
const periodId = default_period._id;
// Then: filter everything by periodId
```

This pattern appears in every controller. If something looks wrong or missing, always check if the period is correct.

---

## Real-Time Sync

- `watchStores(io, userSockets)` — Change Stream on `Store`
- `watchGroups(io, userSockets)` — Change Stream on `Group`
- `watchTeachingPeriods(io, userSockets)` — Change Stream on `TeachingPeriod`
- `watchStoreSettings(io, userSockets)` — Change Stream on `StoreSettings`

All push live updates to connected clients.

---

## Helper: `getStoreIdByModel`

Located at `helpers/getStoreIdByModel`:
A utility to look up the `store_id` for any document, used for authorization checks:
- `getStoreIdForClass.js`
- `getStoreIdForCourse.js`
- `getStoreIdForStudent.js`
- `getStoreIdForTeacher.js`
- etc.

These helpers ensure a user can only modify documents belonging to their own store.
