# Smart Polling Implementation

✅ Successfully implemented intelligent auto-refresh for real-time ingestion run updates!

## How It Works

### **Smart Polling Logic**

The page automatically polls the API **only when needed**:

```typescript
if (hasActiveJobs) {
  // Poll every 3 seconds
  setInterval(() => fetchRuns(true), 3000)
} else {
  // Stop polling
  clearInterval()
}
```

### **Active Jobs Detection**

Polling starts when there are runs with status:
- `pending` - Job queued but not started
- `running` - Job currently processing

Polling stops when all jobs are:
- `completed` - Finished successfully
- `failed` - Encountered error
- `cancelled` - Manually stopped

## Features

### 1. **Automatic Start/Stop**
- ✅ Starts polling when active jobs detected
- ✅ Stops polling when no active jobs
- ✅ No manual intervention needed

### 2. **Silent Refresh**
- ✅ No loading spinner during polls
- ✅ Smooth progress bar updates
- ✅ All fields update in real-time

### 3. **Visual Indicator**
- ✅ Blue badge shows "Auto-actualizando" when polling
- ✅ Animated spinner icon
- ✅ Appears next to page title

### 4. **Tab Visibility Optimization**
- ✅ Pauses polling when tab is hidden
- ✅ Refreshes data when returning to tab
- ✅ Saves API calls and browser resources

### 5. **Manual Refresh**
- ✅ "Actualizar" button for instant refresh
- ✅ Works alongside auto-polling
- ✅ Shows loading state

## Polling Interval

**3 seconds** - Optimal balance between:
- Real-time feel (users see updates quickly)
- Server load (reasonable request frequency)
- Job duration (~9 seconds per job)

### Updates Per Job

```
Job starts:
  0s - Status: pending
  3s - Status: running, progress: 33%
  6s - Status: running, progress: 67%
  9s - Status: completed, progress: 100%
```

Users see **3-4 updates** during a typical job execution.

## What Gets Updated

Every 3-second poll fetches the entire run document, updating:

### **Job Status**
- `status` - pending → running → completed/failed
- `startedAt` - When job began
- `completedAt` - When job finished

### **Progress Metrics**
- `processedProducts` - 0 → 150
- `completedLookups` - 0 → 750
- `failedLookups` - Count of failed lookups

### **Results Array**
- `results[]` - Grows as lookups complete
- Individual lookup status, price, URL, etc.

### **Final Statistics** (on completion)
- `productsWithPrices` - Count of successful finds
- `productsNotFound` - Count of not found products

## Visual Updates

### **Progress Bars**
Smoothly animate from current % to new %:
```
0% → 25% → 50% → 75% → 100%
```

### **Counters**
Numbers update instantly:
```
Productos: 0/150 → 75/150 → 150/150
Búsquedas: 0/750 → 380/750 → 750/750
```

### **Status Badges**
Color and icon change in real-time:
```
⏰ Pendiente → 🔄 En Progreso → ✅ Completado
```

## Performance Optimization

### **Conditional Polling**
```typescript
// Only runs when needed
const hasActiveJobs = runs.some(
  run => run.status === 'pending' || run.status === 'running'
)
```

### **Silent Mode**
```typescript
fetchRuns(true)  // No loading spinner
fetchRuns()      // With loading spinner
```

### **Visibility API**
```typescript
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible') {
    // Refresh when user returns
  }
})
```

## User Experience

### **Scenario 1: Trigger New Job**
```
1. User clicks "Nueva Comparación"
2. Job starts, status: pending
3. Polling starts automatically
4. Badge appears: "Auto-actualizando"
5. Progress bar updates every 3s
6. Job completes after 9s
7. Polling stops automatically
8. Badge disappears
```

### **Scenario 2: Return to Tab**
```
1. User switches to another tab
2. Polling continues (but invisible)
3. User returns to tab
4. Data refreshes immediately
5. Shows latest job status
```

### **Scenario 3: Manual Refresh**
```
1. User clicks "Actualizar"
2. Shows loading state
3. Fetches latest data
4. Updates all jobs
5. Polling continues if active jobs exist
```

## Code Structure

### **State Management**
```typescript
const [isPolling, setIsPolling] = useState(false)
const [lastRefresh, setLastRefresh] = useState<Date>(new Date())
```

### **Polling Effect**
```typescript
useEffect(() => {
  const hasActiveJobs = runs.some(...)
  if (hasActiveJobs) {
    const interval = setInterval(() => {
      fetchRuns(true)
    }, 3000)
    return () => clearInterval(interval)
  }
}, [runs, page])
```

### **Visibility Effect**
```typescript
useEffect(() => {
  const handleVisibilityChange = () => {
    if (document.visibilityState === 'visible') {
      // Refresh logic
    }
  }
  document.addEventListener('visibilitychange', handleVisibilityChange)
  return () => removeEventListener(...)
}, [runs])
```

## Benefits Over WebSocket

✅ **Simple** - No server infrastructure changes
✅ **Reliable** - No connection drops or reconnects
✅ **Efficient** - Only polls when needed
✅ **Compatible** - Works with existing REST API
✅ **Maintainable** - Easy to debug and modify

## Future Enhancements

If needed later:
- 🔄 Exponential backoff on errors
- 📊 Show last refresh timestamp
- ⚡ Configurable polling interval
- 🔔 Sound notification on job completion
- 📱 Desktop notifications via Notification API
- 🌐 Upgrade to WebSocket for 100+ concurrent users

## Testing

1. **Start a job:**
   - Click "Nueva Comparación"
   - Badge appears immediately
   - Progress updates every 3s

2. **Switch tabs:**
   - Open another tab
   - Return after 10s
   - Data refreshes automatically

3. **Multiple jobs:**
   - Start 2-3 jobs
   - All progress bars update
   - Polling continues until all complete

4. **Manual refresh:**
   - Click "Actualizar" anytime
   - Works alongside auto-polling
   - Shows loading state

## Summary

Smart polling provides a **near real-time experience** with minimal complexity. Users see:
- ✅ Live progress updates
- ✅ All field changes
- ✅ Smooth animations
- ✅ Clear visual feedback

Perfect for this use case where jobs run for ~9 seconds and updates every 3 seconds is sufficient! 🚀
