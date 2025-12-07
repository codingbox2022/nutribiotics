# Ingestion Runs UI Implementation

✅ Successfully created a complete UI for viewing and managing ingestion runs!

## What Was Built

### 1. API Integration
**File:** [api.ts](frontend/src/services/api.ts)

Added `ingestionRunsApi` service with methods:
- `getAll(page, limit)` - Paginated list of runs
- `getRecent(limit)` - Recent runs
- `getByStatus(status)` - Filter by status
- `getById(id)` - Get specific run details
- `cancel(id)` - Cancel running/pending jobs

TypeScript interfaces:
- `IngestionRun` - Complete run data structure
- `LookupResult` - Individual product lookup result
- `IngestionRunsResponse` - Paginated response

### 2. Ingestion Runs Page
**File:** [IngestionRuns.tsx](frontend/src/pages/IngestionRuns.tsx)

**Features:**
- ✅ Full list of ingestion runs with pagination
- ✅ Status badges with icons (pending, running, completed, failed, cancelled)
- ✅ Real-time progress bars showing completion percentage
- ✅ Duration tracking (auto-calculated)
- ✅ Product and lookup counters
- ✅ Success metrics (products with prices, not found)
- ✅ Error viewing for failed runs
- ✅ Cancel functionality for pending/running jobs
- ✅ Refresh button to reload data
- ✅ Stats cards showing count by status
- ✅ Loading states with skeleton
- ✅ Empty state when no runs

**Table Columns:**
1. **Estado** - Status badge with icon
2. **Iniciado** - Started date/time
3. **Duración** - Time elapsed (auto-formatted)
4. **Progreso** - Visual progress bar + percentage
5. **Productos** - Processed / Total products
6. **Búsquedas** - Completed / Total lookups (with failed count)
7. **Éxito** - Products with prices / not found (completed runs only)
8. **Acciones** - Cancel button or View Error button

### 3. Navigation
**Updated Files:**
- [Sidebar.tsx](frontend/src/components/layout/Sidebar.tsx) - Added "Procesos" menu item
- [App.tsx](frontend/src/App.tsx) - Added route `/ingestion-runs`

### 4. UI Components
**Updated:** [badge.tsx](frontend/src/components/ui/badge.tsx)
- Added `success` variant for completed status badges

## Status Badges

Each status has a unique visual style:

| Status | Color | Icon | Label |
|--------|-------|------|-------|
| `pending` | Gray (secondary) | Clock | Pendiente |
| `running` | Blue (default) | Loader2 (spinning) | En Progreso |
| `completed` | Green (success) | CheckCircle | Completado |
| `failed` | Red (destructive) | XCircle | Fallido |
| `cancelled` | Outline | Ban | Cancelado |

## Progress Calculation

Progress is calculated as:
```typescript
(processedProducts / totalProducts) × 100
```

Visual progress bar updates in real-time as jobs process.

## Duration Formatting

Auto-formats duration based on time elapsed:
- Less than 1 minute: `45s`
- Less than 1 hour: `5m 30s`
- Over 1 hour: `2h 15m`

## Features Demonstration

### Stats Cards
Shows count of runs by status at the top:
```
┌─────────────┬─────────────┬──────────────┬─────────┬───────────┐
│  Pendiente  │ En Progreso │  Completado  │ Fallido │ Cancelado │
│      2      │      1      │      45      │    3    │     1     │
└─────────────┴─────────────┴──────────────┴─────────┴───────────┘
```

### Actions
- **Cancel** - Available for `pending` and `running` jobs
- **View Error** - Available for `failed` jobs (shows toast with error message)

### Pagination
Bottom navigation to browse through pages:
```
← Anterior | Página 1 de 5 | Siguiente →
```

## Data Flow

```
User visits /ingestion-runs
         ↓
Component calls ingestionRunsApi.getAll(page, limit)
         ↓
API fetches from GET /ingestion-runs?page=1&limit=20
         ↓
Backend returns paginated results
         ↓
Component renders table with all runs
         ↓
User clicks "Cancelar" on running job
         ↓
API calls POST /ingestion-runs/{id}/cancel
         ↓
Job status updates to "cancelled"
         ↓
Component refreshes list
```

## Testing

### 1. View All Runs
```
Navigate to: http://localhost:5173/ingestion-runs
```

### 2. Trigger a New Run
```
Go to: Comparación page
Click: "Comenzar comparación de precios"
Return to: Procesos page
See: New run in "En Progreso" status
```

### 3. Watch Progress
The page auto-updates when you refresh. You'll see:
- Progress bar advancing
- Duration increasing
- Products/lookups counters updating

### 4. View Completed Run
After ~9 seconds, the run completes and shows:
- Status badge changes to "Completado" (green)
- Progress bar at 100%
- Final duration displayed
- Products with prices count
- Products not found count

### 5. Cancel a Job
For any pending/running job:
- Click "Cancelar" button
- Toast notification confirms cancellation
- Status updates to "Cancelado"

### 6. View Error Details
For failed runs:
- Click "Ver Error" button
- Toast shows error message and description

## Responsive Design

The page is fully responsive:
- **Desktop**: Full table with all columns
- **Tablet**: Scrollable table horizontally
- **Mobile**: Stats cards stack vertically, table scrolls

## Color Coding

- 🟢 **Green** - Success (completed runs, products with prices)
- 🔵 **Blue** - In progress (running jobs)
- 🔴 **Red** - Errors (failed jobs, failed lookups)
- ⚪ **Gray** - Neutral (pending jobs, not found products)
- ⚫ **Outline** - Cancelled jobs

## Future Enhancements

Potential improvements:
- Auto-refresh every 5-10 seconds for running jobs
- Expandable rows to view detailed lookup results
- Filter dropdown to show only specific statuses
- Search by triggered date range
- Export results to CSV
- View individual lookup details modal
- Retry failed jobs
- Bulk cancel operations
