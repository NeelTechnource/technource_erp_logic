# DashboardController - Technical Documentation

## Overview

The `DashboardController` is a comprehensive Laravel API controller that serves as the central hub for employee time tracking, attendance management, and task monitoring in the ERP system. It implements sophisticated business logic for different user types, time validation, and attendance workflows.

**File:** `/app/Http/Controllers/Api/v1/Dashboard/DashboardController.php`  
**Namespace:** `App\Http\Controllers\Api\v1\Dashboard`  
**Route:** `GET /api/v1/dashboard`  
**Authentication:** Required  
**Middleware:** `auth:api`

---

## API Endpoint Details

### **Primary Endpoint**
```http
GET /api/v1/dashboard
Authorization: Bearer {token}
Content-Type: application/json
```

### **Response Structure**
```json
{
    "success": true,
    "message": "Dashboard data retrieved successfully",
    "data": {
        "name_section": {
            "full_name": "John Doe",
            "class": "morning",
            "greeting": "Good Morning!"
        },
        "in_progress_section": {
            "in_progress_section_show": 1,
            "list": [...],
            "count": "03"
        },
        "yesterday_log_error": 0,
        "today_working_hours_section": {
            "logged_hours": "8h 30m",
            "biometric_hours": "8h 45m",
            "logged_hours_show": 1,
            "biometric_hours_show": 1,
            "show_full_section": 1,
            "is_error": 0,
            "is_show_text_message": true,
            "text_message": "Hours matched successfully",
            "biometric_error": 0
        },
        "yesterday_working_hours_section": {...},
        "weekly_working_hours_section": {...},
        "previous_week_working_hours_section": {...},
        "monthly_working_hours_section": {...},
        "attendance_section": {
            "attendance_section_show": 1,
            "access_attendance": true,
            "data": {
                "data": [...],
                "current_page": 1,
                "total": 25,
                "per_page": 100
            }
        }
    }
}
```

---

## Database Schema Requirements

### **Core Tables**
```sql
-- Users table
users (
    user_id INT PRIMARY KEY,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    full_name VARCHAR(255),
    email VARCHAR(255),
    user_type TINYINT, -- 1=Employee, 2=Manager, etc.
    timezone VARCHAR(50),
    user_status TINYINT,
    profile_image VARCHAR(255)
);

-- User information extended
user_infos (
    user_info_id INT PRIMARY KEY,
    user_id INT FOREIGN KEY,
    working_hours DECIMAL(4,2) DEFAULT 8.5,
    need_to_log_hours TINYINT, -- 1=Yes, 2=No
    is_remote TINYINT, -- 1=Yes, 2=No
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Attendance records
attendances (
    attendance_id INT PRIMARY KEY,
    user_id INT FOREIGN KEY,
    type TINYINT, -- 1=Leave, 2=TimeAdj, 3=ExtraWFH, 4=Extra, 5=Leave
    status TINYINT, -- 1=Pending, 2=Approved, 3=Rejected
    start_date DATE,
    end_date DATE,
    hours INT,
    minutes INT,
    is_half_day TINYINT, -- 1=No, 2=FirstHalf, 3=SecondHalf, 4=Custom
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Biometric punch records
biometrics (
    biometric_id INT PRIMARY KEY,
    user_id INT FOREIGN KEY,
    machine_name TINYINT, -- 1,3=In, 2,4=Out
    punch_timestamp TIMESTAMP,
    status TINYINT, -- 1=Success, 2=Error
    created_at TIMESTAMP
);

-- Project task work logs
project_task_work_logs (
    work_log_id INT PRIMARY KEY,
    user_id INT FOREIGN KEY,
    project_task_id INT FOREIGN KEY,
    work_log_date DATE,
    time_spent_in_hour INT,
    time_spent_in_minute INT,
    description TEXT,
    created_at TIMESTAMP
);
```

---

## Technical Implementation Details

### **1. Authentication & Authorization**
```php
// Authentication check
$user = auth()->user();
if (!$user) {
    return $this->errorResponse('Unauthorized', 401);
}

// Permission checks for attendance section
$canViewAttendance = $user->can('attendance.index') || $user->can('attendance.show');
```

### **2. Timezone Handling**
```php
// Get user timezone with fallback
$timezone = $user->timezone ?? config('app.timezone');
$currentHour = now()->setTimezone($timezone)->format('H');

// Time-based greeting logic
$greeting = match(true) {
    $currentHour >= 6 && $currentHour < 12 => ['class' => 'morning', 'text' => 'Good Morning!'],
    $currentHour >= 12 && $currentHour < 16 => ['class' => 'afternoon', 'text' => 'Good Afternoon!'],
    $currentHour >= 16 && $currentHour < 20 => ['class' => 'evening', 'text' => 'Good Evening!'],
    default => ['class' => 'night', 'text' => 'Good Evening!']
};
```

### **3. Complex Query Examples**

#### **Holiday and Working Day Calculation**
```php
// Get system holidays (excluding additional working days)
$holidays = HolidayEtc::whereNot('type', 3)
    ->whereNull('deleted_at')
    ->pluck('date')
    ->map(fn($date) => Carbon::parse($date)->format('Y-m-d'))
    ->toArray();

// Get holidays converted to working days
$additionalWorkingDays = HolidayEtc::where('type', 3)
    ->whereNotNull('additional_date')
    ->whereNull('deleted_at')
    ->pluck('additional_date')
    ->map(fn($date) => Carbon::parse($date)->format('Y-m-d'))
    ->toArray();

// Smart working day calculation
$lastWorkingDay = Carbon::yesterday();
while ($this->isNonWorkingDay($lastWorkingDay, $holidays, $additionalWorkingDays, $userLeaveDates)) {
    $lastWorkingDay->subDay();
}
```

#### **Attendance Prioritization Query**
```sql
SELECT *
FROM attendances a
WHERE a.status IN (1, 5) -- Pending and Cancel Request
  AND EXISTS (
    SELECT 1 FROM user_leave_approvers ula 
    WHERE ula.user_id = a.user_id AND ula.approver_id = ?
  )
ORDER BY 
  -- Priority 1: Overdue requests
  CASE 
    WHEN (a.type IN (3,5) AND a.start_date < CURDATE() AND a.end_date < CURDATE() AND a.status IN (1,5)) THEN 1
    WHEN (a.type NOT IN (3,5) AND a.start_date < CURDATE() AND a.status IN (1,5)) THEN 1
    ELSE 2
  END,
  -- Priority 2: Current date requests  
  CASE
    WHEN (a.type IN (3,5) AND CURDATE() BETWEEN a.start_date AND a.end_date) THEN 1
    WHEN (a.type NOT IN (3,5) AND CURDATE() = a.start_date) THEN 1
    ELSE 2
  END,
  a.start_date ASC
LIMIT 100;
```

### **4. Time Calculation Algorithms**

#### **Biometric Time Calculation**
```php
private function calculateBiometricTime($punchRecords): array
{
    $totalMinutes = 0;
    $entryTime = null;
    $records = $punchRecords->orderBy('biometric_id')->get();
    
    foreach ($records as $record) {
        if (empty($record->punch_timestamp)) continue;
        
        if (in_array($record->machine_name, [1, 3])) { // Punch In
            $entryTime = Carbon::parse($record->punch_timestamp);
        } elseif (in_array($record->machine_name, [2, 4]) && $entryTime) { // Punch Out
            $exitTime = Carbon::parse($record->punch_timestamp);
            $totalMinutes += $exitTime->diffInMinutes($entryTime);
            $entryTime = null;
        }
    }
    
    // Handle incomplete sessions (still punched in today)
    if ($entryTime && $entryTime->isToday()) {
        $totalMinutes += Carbon::now()->diffInMinutes($entryTime);
    }
    
    return [
        'minutes' => $totalMinutes,
        'formatted' => sprintf('%dh %dm', floor($totalMinutes / 60), $totalMinutes % 60)
    ];
}
```

#### **Working Hours Validation Logic**
```php
private function validateWorkingHours($loggedMinutes, $biometricMinutes, $expectedMinutes, $isYesterday = false): array
{
    $isError = 0;
    $message = 'Hours matched successfully';
    
    // Primary validation: Over-logging check
    if ($loggedMinutes > $biometricMinutes) {
        $isError = 1;
        $message = 'Logged hours exceed biometric hours';
    }
    // Yesterday-specific validation
    elseif ($isYesterday) {
        if ($loggedMinutes == 0 && $expectedMinutes == 0) {
            // Holiday or leave day - no error
            $isError = 0;
        } elseif ($loggedMinutes < $expectedMinutes) {
            // Under-logging for completed work day
            $isError = 1;
            $message = 'Insufficient hours logged for work day';
        }
    }
    
    return ['is_error' => $isError, 'message' => $message];
}
```

---

## Advanced Features Implementation

### **1. Multi-User Type Support**
```php
class UserTypeStrategy
{
    public static function getDisplayRules(User $user, UserInfo $userInfo): array
    {
        return match($user->user_type) {
            User::USER_TYPE_EMPLOYEE => [
                'show_logged_hours' => $userInfo?->need_to_log_hours === UserInfo::NEED_TO_LOG_HOUR_YES,
                'show_biometric_hours' => $userInfo?->is_remote !== UserInfo::IS_REMOTE_YES,
                'show_tasks' => $userInfo?->need_to_log_hours === UserInfo::NEED_TO_LOG_HOUR_YES,
                'validate_hours' => true
            ],
            User::USER_TYPE_MANAGER => [
                'show_logged_hours' => false,
                'show_biometric_hours' => true,
                'show_tasks' => false,
                'validate_hours' => false
            ],
            default => [
                'show_logged_hours' => false,
                'show_biometric_hours' => false,
                'show_tasks' => false,
                'validate_hours' => false
            ]
        };
    }
}
```

### **2. Dynamic Date Range Processing**
```php
private function processDateRange($dateRange): array
{
    if (is_array($dateRange)) {
        // Handle date ranges for weekly/monthly calculations
        [$startDate, $endDate] = $dateRange;
        return [
            'start' => Carbon::parse($startDate)->startOfDay(),
            'end' => Carbon::parse($endDate)->endOfDay(),
            'type' => 'range'
        ];
    } else {
        // Handle single date for daily calculations
        return [
            'date' => Carbon::parse($dateRange)->startOfDay(),
            'type' => 'single'
        ];
    }
}

private function applyDateFilters($query, $dateRange)
{
    $processed = $this->processDateRange($dateRange);
    
    if ($processed['type'] === 'range') {
        return $query->whereBetween('created_at', [$processed['start'], $processed['end']]);
    } else {
        return $query->whereDate('created_at', $processed['date']);
    }
}
```

### **3. Resource Collection Implementation**
```php
// AttendanceCollection.php
class AttendanceCollection extends ResourceCollection
{
    public function toArray($request): array
    {
        return [
            'data' => $this->collection->map(function ($attendance) {
                return [
                    'id' => $attendance->attendance_id,
                    'user' => [
                        'id' => $attendance->user->user_id,
                        'name' => $attendance->user->full_name,
                        'profile_image' => $attendance->user->profile_image
                    ],
                    'type' => $this->getTypeLabel($attendance->type),
                    'status' => $this->getStatusLabel($attendance->status),
                    'dates' => [
                        'start' => $attendance->start_date,
                        'end' => $attendance->end_date
                    ],
                    'duration' => $this->calculateDuration($attendance),
                    'priority' => $this->calculatePriority($attendance),
                    'created_at' => $attendance->created_at->format('Y-m-d H:i:s')
                ];
            }),
            'meta' => [
                'total' => $this->total(),
                'per_page' => $this->perPage(),
                'current_page' => $this->currentPage()
            ]
        ];
    }
}
```

---

## Error Handling & Logging

### **1. Exception Handling**
```php
try {
    $dashboardData = $this->generateDashboardData($user);
    return $this->successResponse($dashboardData, 'Dashboard loaded successfully');
} catch (ValidationException $e) {
    Log::warning('Dashboard validation error', [
        'user_id' => $user->user_id,
        'error' => $e->getMessage()
    ]);
    return $this->errorResponse('Validation error', 422, $e->errors());
} catch (Exception $e) {
    Log::error('Dashboard generation failed', [
        'user_id' => $user->user_id,
        'error' => $e->getMessage(),
        'trace' => $e->getTraceAsString()
    ]);
    return $this->errorResponse('Dashboard temporarily unavailable', 500);
}
```

### **2. Performance Monitoring**
```php
private function generateDashboardData(User $user): array
{
    $startTime = microtime(true);
    
    // Generate dashboard sections
    $data = [
        'name_section' => $this->getNameSection($user),
        'in_progress_section' => $this->getInProgressSection($user, $userInfo),
        // ... other sections
    ];
    
    $executionTime = microtime(true) - $startTime;
    
    Log::info('Dashboard generation completed', [
        'user_id' => $user->user_id,
        'execution_time' => $executionTime,
        'memory_usage' => memory_get_usage(true)
    ]);
    
    return $data;
}
```

---

## Testing Examples

### **1. Unit Test Example**
```php
// Tests/Unit/DashboardControllerTest.php
class DashboardControllerTest extends TestCase
{
    use RefreshDatabase, WithFaker;
    
    public function test_dashboard_returns_correct_structure()
    {
        // Arrange
        $user = User::factory()->create(['user_type' => User::USER_TYPE_EMPLOYEE]);
        $userInfo = UserInfo::factory()->create([
            'user_id' => $user->user_id,
            'need_to_log_hours' => UserInfo::NEED_TO_LOG_HOUR_YES
        ]);
        
        // Act
        $response = $this->actingAs($user)->getJson('/api/v1/dashboard');
        
        // Assert
        $response->assertStatus(200)
            ->assertJsonStructure([
                'success',
                'message',
                'data' => [
                    'name_section' => ['full_name', 'class', 'greeting'],
                    'in_progress_section' => ['in_progress_section_show', 'list', 'count'],
                    'today_working_hours_section' => [
                        'logged_hours', 'biometric_hours', 'is_error'
                    ]
                ]
            ]);
    }
    
    public function test_working_day_calculation_skips_holidays()
    {
        // Arrange
        $yesterday = Carbon::yesterday();
        HolidayEtc::factory()->create([
            'date' => $yesterday->format('Y-m-d'),
            'type' => 1 // Holiday
        ]);
        
        $user = User::factory()->create();
        $controller = new DashboardController();
        
        // Act
        $lastWorkingDay = $this->invokeMethod($controller, 'getLastWorkingDay', [$user]);
        
        // Assert
        $this->assertNotEquals($yesterday->format('Y-m-d'), $lastWorkingDay->format('Y-m-d'));
    }
}
```

### **2. Integration Test Example**
```php
public function test_dashboard_with_biometric_data()
{
    // Arrange
    $user = User::factory()->create();
    
    // Create biometric punch records
    BioMetric::factory()->create([
        'user_id' => $user->user_id,
        'machine_name' => 1, // Punch in
        'punch_timestamp' => Carbon::today()->setTime(9, 0)
    ]);
    BioMetric::factory()->create([
        'user_id' => $user->user_id,
        'machine_name' => 2, // Punch out
        'punch_timestamp' => Carbon::today()->setTime(18, 0)
    ]);
    
    // Create work log
    ProjectTaskWorkLog::factory()->create([
        'user_id' => $user->user_id,
        'work_log_date' => Carbon::today(),
        'time_spent_in_hour' => 8,
        'time_spent_in_minute' => 30
    ]);
    
    // Act
    $response = $this->actingAs($user)->getJson('/api/v1/dashboard');
    
    // Assert
    $response->assertStatus(200)
        ->assertJson([
            'data' => [
                'today_working_hours_section' => [
                    'logged_hours' => '8h 30m',
                    'biometric_hours' => '9h 0m',
                    'is_error' => 0
                ]
            ]
        ]);
}
```

---

## Performance Optimization

### **1. Query Optimization**
```php
// Optimized queries with eager loading
$attendanceData = Attendance::with([
    'user:user_id,first_name,last_name,user_status,profile_image',
    'user.userLeaveApprover:user_id,approver_id',
    'user.userLeaveViewer:user_id,viewer_id'
])
->select('attendance_id', 'user_id', 'type', 'status', 'start_date', 'end_date')
->whereIn('status', [Attendance::PENDING_STATUS, Attendance::CANCEL_REQUEST_STATUS])
->limit(100);
```

### **2. Caching Strategy**
```php
private function getCachedUserInfo(int $userId): ?UserInfo
{
    $cacheKey = "user_info:{$userId}";
    
    return Cache::remember($cacheKey, 3600, function () use ($userId) {
        return UserInfo::where('user_id', $userId)->first();
    });
}

private function getCachedHolidays(): array
{
    return Cache::remember('system_holidays', 86400, function () {
        return HolidayEtc::whereNot('type', 3)
            ->whereNull('deleted_at')
            ->pluck('date')
            ->map(fn($date) => Carbon::parse($date)->format('Y-m-d'))
            ->toArray();
    });
}
```

### **3. Database Indexing Recommendations**
```sql
-- Critical indexes for performance
CREATE INDEX idx_attendances_user_status ON attendances(user_id, status);
CREATE INDEX idx_attendances_dates ON attendances(start_date, end_date);
CREATE INDEX idx_biometrics_user_timestamp ON biometrics(user_id, punch_timestamp);
CREATE INDEX idx_work_logs_user_date ON project_task_work_logs(user_id, work_log_date);
CREATE INDEX idx_holidays_date_type ON holiday_etcs(date, type);

-- Composite indexes for complex queries
CREATE INDEX idx_attendances_complex ON attendances(user_id, type, status, start_date, end_date);
CREATE INDEX idx_biometrics_machine_time ON biometrics(user_id, machine_name, punch_timestamp);
```

---

## Security Considerations

### **1. Data Access Control**
```php
// Ensure users only see their own data
$query->where('user_id', auth()->user()->user_id);

// Manager permission checks
$attendanceQuery->whereHas('user.userLeaveApprover', function ($q) {
    $q->where('approver_id', auth()->user()->user_id);
});
```

### **2. Input Validation**
```php
// Custom request validation
class DashboardRequest extends FormRequest
{
    public function authorize(): bool
    {
        return auth()->check();
    }
    
    public function rules(): array
    {
        return [
            'date_filter' => 'sometimes|date_format:Y-m-d',
            'section' => 'sometimes|string|in:tasks,hours,attendance'
        ];
    }
}
```

### **3. Rate Limiting**
```php
// Apply rate limiting to dashboard endpoint
Route::middleware(['auth:api', 'throttle:dashboard'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});

// config/cache.php
'stores' => [
    'rate_limiter' => [
        'dashboard' => '60,1', // 60 requests per minute
    ]
]
```

---

## Deployment Considerations

### **1. Environment Configuration**
```bash
# .env settings
DASHBOARD_CACHE_TTL=3600
DASHBOARD_MAX_RECORDS=100
BIOMETRIC_TIMEOUT=30
TIME_VALIDATION_STRICT=true
```

### **2. Queue Integration**
```php
// For heavy calculations, use queues
dispatch(new CalculateMonthlyHoursJob($user->user_id))->delay(now()->addSeconds(30));

// Job implementation
class CalculateMonthlyHoursJob implements ShouldQueue
{
    public function handle()
    {
        $monthlyData = $this->calculateComplexMonthlyMetrics();
        Cache::put("monthly_hours:{$this->userId}", $monthlyData, 3600);
    }
}
```

---

## API Documentation

### **OpenAPI Specification**
```yaml
/api/v1/dashboard:
  get:
    summary: Get dashboard data
    tags: [Dashboard]
    security:
      - BearerAuth: []
    responses:
      200:
        description: Dashboard data retrieved successfully
        content:
          application/json:
            schema:
              type: object
              properties:
                success:
                  type: boolean
                  example: true
                message:
                  type: string
                  example: "Dashboard data retrieved successfully"
                data:
                  $ref: '#/components/schemas/DashboardData'
      401:
        description: Unauthorized
      500:
        description: Server error

components:
  schemas:
    DashboardData:
      type: object
      properties:
        name_section:
          $ref: '#/components/schemas/NameSection'
        in_progress_section:
          $ref: '#/components/schemas/TaskSection'
        today_working_hours_section:
          $ref: '#/components/schemas/WorkingHoursSection'
```

This technical documentation provides developers with comprehensive implementation details, code examples, testing approaches, and deployment considerations for maintaining and extending the DashboardController functionality.
