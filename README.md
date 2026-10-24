# Frontistirio — Store Management Brain

> **Κατάσταση τεκμηρίωσης** (σφραγίδα 2026-10-24) · τελευταία γραφή **2026-09-21**.
> **Δεν φέρει επαλήθευση έναντι κώδικα** — κανόνας #4 του ευρετηρίου. Από εκείνη την ημερομηνία ο κώδικας
> έχει προχωρήσει **89 commits** στο frontend και **20** στο API (όλο το repo, όχι μόνο αυτό το feature).
> Σημερινά HEAD: frontend `0f466e0` / API `cabaa2c`.
> Η γραφή της 2026-09-21 πρόσθεσε **μόνο παραπομπές**· το σώμα σταματά **2026-07-06**.
> Τι είναι **γνωστό** ότι λείπει: §5 και §5.1 του ευρετηρίου (`Frontistirio-brain-repos-index`).

> **Full-stack reference** for stores, groups, teaching periods, store settings, and classrooms — the top-level organizational layer. The teaching period is the most critical concept: it scopes ALL data throughout the system.
>
> **See also (2026-09-21):** the `/settings` page is now a hub (`.sp-*`, per-row autosave), and per-store **branding** lives in
> `StoreSettings.branding`. Both are documented in **`Frontistirio-ui-conventions-brain-repo`** §3 and §6. Store subscriptions are in the
> superadmin brain repo. ⚠️ This README was last verified 2026-07-06.

---

## Organizational Hierarchy

```
Group (franchise/chain)  ← superadmin manages
  └── Store (individual tutoring center) ← store-user manages
        ├── TeachingPeriod (one is always "default" = active) ← THE scoping key
        ├── StoreSettings (configuration)
        └── All entities (students, teachers, courses, classes, materials...)
              → All period-scoped via { period: ObjectId, ... } arrays
```

---

## Backend: MongoDB Models

### `models/stores.js`
```javascript
{
  name: String,
  group: ObjectId → Group,
  address: ObjectId → Address,
  user_id: ObjectId → User,
  teaching_periods: [ObjectId → TeachingPeriod],
  isDeleted: Boolean, deletedAt: Date,
  createdBy/updatedBy: ObjectId → User,
  timestamps: true
}
```

### `models/group.js`
```javascript
{ name: String, stores: [ObjectId → Store] }
```

### `models/teaching_periods.js` — THE CENTRAL SCOPING MODEL
```javascript
{
  name: String,
  date_from: Date (required),
  date_to: Date (required),
  store_id: ObjectId → Store,
  default: Boolean (default: false),  // ← EXACTLY ONE should be true per store
  updatedBy: ObjectId → User,
  isDeleted: Boolean, deletedAt: Date,
  timestamps: true
}
```

### `models/store-settings.js`
```javascript
{
  store_id: ObjectId → Store,
  course_hour_duration: Number (required),   // teaching-hour length in minutes
  break_times: [{ dayOfWeek: 1-7, timeRanges: [{ startTime: "HH:mm", endTime: "HH:mm" }] }],
  require_grade_approval: Boolean (default false),  // added 2026-07-06 — teacher grades need admin approval
  createdBy/updatedBy: ObjectId → User, isDeleted, deletedAt, timestamps
}
```

**Adding a new setting field:** the upsert controller (`upsertStoreSettings`) copies every key from
`req.body.settings` onto the doc, but `updateStoreSettings` only persists keys in
`ALLOWED_UPDATE_FIELDS` (`controllers/handlers/store-settings.js`). So a new field needs BOTH the schema
field AND an entry in `ALLOWED_UPDATE_FIELDS` (currently
`["course_hour_duration", "break_times", "require_grade_approval"]`). The frontend `SetCourseSettingsComponent`
sends the whole `store_settings` object, so a simple `ion-toggle`/input bound to the field is enough on the UI side.
`require_grade_approval` powers the grade-approval workflow — see the `grading-system` brain.

### `models/address.js`
```javascript
{ address: String, number: Number, zip_code: Number, country: String }
```

### `models/classroom.js`
```javascript
{ name: String, capacity: Number, store_id: ObjectId → Store }
```

---

## Backend: Routes

### `/stores` (`routes/stores.js`)
| Method | Path | Role | Controller |
|---|---|---|---|
| POST | `/stores/create-store` | superadmin | `createStore` |
| POST | `/stores/edit-store` | superadmin | `editStore` |
| DELETE | `/stores/delete-store` | superadmin | `deleteStore` |
| GET | `/stores/getAll` | superadmin | `getAllStores` |

### `/group` (`routes/group.js`)
| Method | Path | Role |
|---|---|---|
| POST | `/group/create-group` | superadmin |
| POST | `/group/edit-group` | superadmin |
| DELETE | `/group/delete-group` | superadmin |
| GET | `/group/getAll` | superadmin |

### `/teaching-periods` (`routes/teaching_periods.js`)
| Method | Path | Role | Controller |
|---|---|---|---|
| GET | `/teaching-periods/get-teaching-periods` | isStoreUser | `getStoreTeachingPeriods` |
| POST | `/teaching-periods/get-store-teaching-period` | isStoreUser | `getStoreTeachingPeriodReferences` |
| POST | `/teaching-periods/get-store-teaching-period-classes` | isStoreUser | `getStoreTeachingPeriodClasses` |
| POST | `/teaching-periods/get-store-teaching-period-students` | isStoreUser | `getStoreTeachingPeriodStudents` |
| POST | `/teaching-periods/create-store-teaching-period` | isStoreUser | `createStoreTeachingPeriod` |
| POST | `/teaching-periods/edit-store-teaching-period` | isStoreUser | `editStoreTeachingPeriod` |
| DELETE | `/teaching-periods/delete-store-teaching-period` | isStoreUser | `deleteStoreTeachingPeriod` |
| POST | `/teaching-periods/set-default-teaching-period` | isStoreUser | `setDefaultTeachingPeriod` |

### `/store-settings` (`routes/store-settings.js`)
| Method | Path | Role |
|---|---|---|
| GET | `/store-settings/get-store-settings` | isStoreUser |
| POST | `/store-settings/upsert-store-settings` | isStoreUser |

### `/classrooms` (`routes/classrooms.js`)
| Method | Path | Role |
|---|---|---|
| GET | `/classrooms/get-store-classrooms` | isStoreUser |
| POST | `/classrooms/create-store-classroom` | isStoreUser |
| POST | `/classrooms/edit-store-classroom` | isStoreUser |
| DELETE | `/classrooms/delete-store-classroom` | isStoreUser |

---

## Backend: Store Creation (Controller — MongoDB Transaction)

```javascript
// body: { name, address: { address, number, zip_code, country }, group: groupId }
const new_address = await addressHandler.createAddress(
  address.address, address.number, address.zip_code, address.country, req.user.id, session
);
const new_store = await storeHandler.createStore(
  body.name, group_id, new_address._id, req.user.id, session
);
// Bidirectional: add store to group
await groupsHandler.updateGroupStores(group_id, new_store._id, session);
await session.commitTransaction();
```

---

## Backend: Teaching Period System

### `getStoreDefaultPeriod(storeId, session?)` — used EVERYWHERE
```javascript
return await TeachingPeriod.findOne({ store_id: storeId, default: true });
```
This single query is called at the start of virtually every controller that handles store-specific data. If it returns `null`, most controllers throw `'not_set_default_period'` or similar.

### `setDefaultTeachingPeriod(periodId, store_id, session)` — global scope change
```javascript
// Step 1: Unset ALL other periods for this store
await TeachingPeriod.updateMany(
  { store_id: store_id, _id: { $ne: periodId } },
  { $set: { default: false } },
  { session }
);
// Step 2: Set the target period as default
period.default = true;
await period.save({ session });
```

**Warning:** Calling this changes the active period for the entire store. Everything the frontend shows immediately shifts to the new period's data. All bootstrap data becomes stale.

### `createStoreTeachingPeriod` (Controller — Transaction)
1. Validate `date_from`, `date_to`
2. Create `TeachingPeriod` document
3. Call `storeHandler.updateStoreTeachingPeriods(store_id, [period._id], user_id, session)` — adds the period to `Store.teaching_periods`
4. Commit transaction

### Handler Exports
```javascript
exports.createStoreTeachingPeriods(teachingPeriodData, store_id, user_id, session)
exports.fetchStoreTeachingPeriods(store_id)
exports.editStoreTeachingPeriod(teachingPeriodData, store_id, user_id)
exports.deleteStoreTeachingPeriod(periodId, permanently, user_id, session)
exports.getStoreTeachingPeriod(periodId)
exports.setDefaultTeachingPeriod(periodId, store_id, session)
exports.getStoreDefaultPeriod(storeId, session?)    // ← called everywhere
exports.watchTeachingPeriods(io, userSockets)
```

---

## Frontend: NgRx Slices for Store Management

```typescript
AppState {
  groups: GroupState              // { groups: Group[] }
  stores: StoresState             // { stores: Store[] }
  teaching_periods: TeachingPeriodsState  // { teaching_periods: TeachingPeriod[] }
  classrooms: ClassroomsState     // { classrooms: Classroom[] }
  storeSettings: StoreSettingsState   // { settings: StoreSettings | null }
}
```

Selectors: `selectAllGroups`, `selectStores`, `selectTeachingPeriods`, `selectClassRooms`, `selectStoreSettings`

---

## Frontend: The Default Period Pattern (Critical)

The active period is found reactively everywhere in the frontend:
```typescript
// In any component:
this.store.select(selectTeachingPeriods).pipe(
  map(periods => periods.find(p => p.default === true))
)
```

`HomePage` uses `combineLatest` including `selectTeachingPeriods` — when the default period changes (socket `periodUpdated` event → NgRx update), the home page re-renders with the new period's data.

---

## Frontend: Teaching Periods Service (`services/teaching-periods.service.ts`)

```typescript
addTeachingPeriod(teaching_period)
  → POST /teaching-periods/create-store-teaching-period

editTeachingPeriod(teaching_period)
  → POST /teaching-periods/edit-store-teaching-period

setDefaultPeriod(periodId)
  → POST /teaching-periods/set-default-teaching-period
  // ⚠️ Changes what ALL components show — triggers re-render of all period-scoped data

deleteTeachingPeriod(periodId, permanently)
  → DELETE /teaching-periods/delete-store-teaching-period

getStoreTeachingPeriodCourses(periodId)
  → POST /teaching-periods/get-store-teaching-period  body: { periodId }

getStoreTeachingPeriodClasses(periodId)
  → POST /teaching-periods/get-store-teaching-period-classes

getStoreTeachingPeriodStudents(periodId)
  → POST /teaching-periods/get-store-teaching-period-students
```

---

## Frontend: Settings Service (`services/settings.service.ts`)

```typescript
getStoreSettings()      → GET /store-settings/get-store-settings
updateStoreSettings(s)  → POST /store-settings/upsert-store-settings
```

---

## Frontend: Classroom Service (`services/classroom.service.ts`)

```typescript
getStoreClassrooms()        → GET /classrooms/get-store-classrooms
addClassroom(room)          → POST /classrooms/create-store-classroom
editClassroom(room)         → POST /classrooms/edit-store-classroom
deleteClassroom(id, perm)   → DELETE /classrooms/delete-store-classroom
```

---

## Frontend: Components

### `GroupsComponent` (`/groups`) + `AddGroupComponent` + `GroupItemComponent`
Superadmin only. Lists groups with their stores. Create/edit groups. Each group shows its stores.

### `StoresComponent` (`/stores`) + `StoreDetailsComponent` (`/store/:id`)
Superadmin only. `StoreDetailsComponent` shows store info + list of users. Contains `AddStoreUserComponent` — creates a store-user account for the store.

### `TeachingPeriodsComponent` (`/periods`, role: store-user)
Lists all periods for the store. Shows which is currently default.

### `TeachingPeriodItemComponent`
The richest period component (8.7KB TS):
- View period details
- "Set as Default" button → `setDefaultPeriod(periodId)` → changes global data scope
- Import data from another period (copy-forward functionality)
- Shows students, courses, classes within this period

### `SettingsComponent` (`/settings`)
General store configuration. Contains `SetCourseSettingsComponent` for per-course settings.

### `ClassroomsComponent` (`/classrooms`, role: store-user)
Lists physical classrooms. `AddClassroomDiagramComponent` (13KB) provides an interactive layout diagram builder for room configuration.

---

## Real-Time Sync

| Backend watcher | Events emitted | NgRx actions |
|---|---|---|
| `watchGroups` | `groupAdded/Updated/Deleted` | `GroupActions.*` |
| `watchStores` | `storeAdded/Updated/Deleted` | `StoresActions.*` |
| `watchTeachingPeriods` | `periodAdded/Updated/Deleted` | `TeachingPeriodsActions.*` |
| `watchClassRooms` | `classRoomsUpdated` (single event) | `ClassroomsActions.*` |
| `watchStoreSettings` | `settingsAdded/Updated` | `StoreSettingsActions.*` |

When a period's `default` field changes via socket (e.g. another user set a different period as default), the frontend NgRx `teaching_periods` array updates, and all components using `selectTeachingPeriods` with `find(p => p.default === true)` automatically re-render with the new period's data.

---

## `getStoreIdByModel` Helpers

Located at `helpers/getStoreIdByModel` — a set of helper functions used by watchers to identify which store a changed document belongs to:

```javascript
// helpers/getStoreIdForStudent.js
module.exports = async (studentId) => {
  const student = await Student.findById(studentId, 'store_id');
  return student?.store_id;
};
// Similar files exist for: Class, Course, Teacher, Grade, Material, Period, Classroom, StoreSettings
```

These are needed for UPDATE events where `store_id` may not be in `updatedFields`. The watcher calls the helper to look up the store, then targets only that store's connected users.
