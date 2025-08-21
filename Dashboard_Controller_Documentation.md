# Dashboard Controller Technical Documentation
## ERP System - Time Tracking & Employee Management

---

## Table of Contents
1. [Overview](#overview)
2. [Main Dashboard Function](#main-dashboard-function)
3. [Function Deep Dive](#function-deep-dive)
4. [Technical Architecture](#technical-architecture)

---

## Overview

The `DashboardController` is the central hub of the ERP system's employee dashboard. It aggregates multiple data sources to provide:

- **Employee Work Status**: Current tasks, work hours, attendance
- **Time Tracking**: Logged hours vs biometric data validation
- **Approval Workflows**: Leave requests and attendance management
- **Personalization**: Time-based greetings and user-specific data

### Key Technologies
- **Laravel Framework**: PHP web application framework
- **Carbon**: Date/time manipulation library
- **Eloquent ORM**: Database relationships and queries
- **Laravel Collections**: Data manipulation and transformation

---

## Main Dashboard Function

### `index(Request $request)` - Dashboard Data Aggregator

**Purpose**: Collects and returns all dashboard data in a single API response

```php
public function index(Request $request)
{
    $user = auth()->user();
    $userInfo = UserInfo::where('user_id', $user->user_id)->first();
    
    // Collect all dashboard sections
    $name_section = $this->getNameSection($user);
    $in_progress_section = $this->getInProgressSection($user, $userInfo);
    $yesterday_log_error = $this->checkYesterdayLogError($user, $userInfo);
    $today_working_hours_section = $this->todaySection($userInfo, $user->user_type == User::USER_TYPE_EMPLOYEE ? 1 : 0);
    $weekly_working_hours_section = $this->weekSection($userInfo, $user->user_type == User::USER_TYPE_EMPLOYEE ? 1 : 0);
    $previous_week_working_hours_section = $this->previousWeekSection($userInfo, $user->user_type == User::USER_TYPE_EMPLOYEE ? 1 : 0);
    $monthly_working_hours_section = $this->monthSection($userInfo, $user->user_type == User::USER_TYPE_EMPLOYEE ? 1 : 0);
    $attendance_section = $this->attendance($request, $user->user_type == User::USER_TYPE_EMPLOYEE ? 1 : 0);
    
    return $this->successResponse($data, trans('admin.name_section_get'));
}
```

---

## Function Deep Dive

### 1. Dynamic Greeting System - `getNameSection($user)`

**Purpose**: Creates personalized, timezone-aware greetings

**Key Features**:
- **Timezone Support**: Uses user's timezone or system default
- **Time Classification**: Morning (6-12), Afternoon (12-16), Evening (16-20), Night (20-6)
- **UI Integration**: Returns CSS class for styling
- **Personalization**: Includes user's full name

```php
$timezone = $user->timezone ?? config('app.timezone');
$hours = now()->setTimezone($timezone)->format('H');

if ($hours >= 6 && $hours < 12) {
    $greeting = 'Good Morning!';
} elseif ($hours >= 12 && $hours < 16) {
    $greeting = 'Good Afternoon!';
} // ... etc
```

### 2. Active Task Management - `getInProgressSection($user, $userInfo)`

**Purpose**: Displays current tasks for employees who track work hours

**Business Rules**:
- Employee only (not managers/admins)
- Must be required to log hours
- Only shows "In Progress" tasks
- Tasks from user's projects only

**Technical Implementation**:
```php
// Get user's project memberships (exclude clients)
$projectIds = ProjectMember::where('user_id', $user->user_id)
    ->whereNot('type', ProjectMember::TEAM_MEMBER_TYPE_CLIENT)
    ->pluck('project_id');

// Find in-progress tasks assigned to user
$tasks = ProjectTask::whereIn('project_id', $projectIds)
    ->whereHas('TaskStatusType', function ($query) {
        $query->where('status_type', TaskStatusType::TASK_STATUS_TYPE_IN_PROGRESS);
    })
    ->where('assigned_user_id', $user->user_id)
    ->get();
```

### 3. Work Validation Engine - `checkYesterdayLogError($user, $userInfo)`

**Purpose**: Validates proper work hour logging for the last working day

**Complex Algorithm**:
1. **Holiday Detection**: Gets company holidays from database
2. **Leave Processing**: Expands date ranges into individual dates
3. **Last Working Day**: Finds previous valid working day
4. **Validation**: Compares logged vs expected hours

**Key Logic**:
```php
// Holiday detection
$holidayDates = HolidayEtc::whereNot('type', 3)
    ->whereNull('deleted_at')
    ->pluck('date')
    ->toArray();

// Last working day algorithm
$lastWorkingDay = Carbon::yesterday();
while (
    in_array($lastWorkingDay->toDateString(), $holidayDates) ||
    in_array($lastWorkingDay->dayOfWeek, [Carbon::SATURDAY, Carbon::SUNDAY]) ||
    in_array($lastWorkingDay->toDateString(), $leaveDates)
) {
    $lastWorkingDay->subDay();
}
```

### 4. Core Time Tracking Engine - `calculateWorkLogAndBiometricHours()`

**Purpose**: Compares logged hours vs actual work hours from biometric devices

**Key Components**:
- **Working Hours Calculation**: Base hours + adjustments
- **Work Log Processing**: Sum of logged project hours
- **Biometric Processing**: Punch in/out time calculation
- **Extra Work Handling**: Overtime and additional work
- **Error Detection**: Identifies time discrepancies

**Working Hours Adjustments**:
```php
$user_hours = $userInfo->working_hours ?? 8.5; // Default 8.5 hours

// Time adjustments for early leave, late arrival
if ($timeAdjustment) {
    $user_hours -= (float) $timeAdjustment;
}

// Half-day leave reduces hours by 4
if ($isHalfDayLeave) {
    $user_hours -= 4;
}
```

**Error Detection Logic**:
```php
if ($loggedHoursInMinutes > $biometricHoursWithExtraWork) {
    $isError = 1; // Time theft: logged more than biometric shows
} else if ($loggedHoursInMinutes < $userInfoWorkingHoursInMinutes) {
    $isError = 1; // Insufficient hours logged
} else {
    $isError = 0; // Hours match expectations
}
```

### 5. Biometric Time Calculation - `punchCalculateTime($punchInOut)`

**Purpose**: Calculates work hours from biometric punch records

**Machine Code Logic**:
- **Codes 1, 3**: Entry/Punch In
- **Codes 2, 4**: Exit/Punch Out
- **Pairing**: Each entry paired with next exit
- **Active Session**: Ongoing work calculated to current time

```php
foreach ($data as $log) {
    if (in_array($log->machine_name, [1, 3])) {
        $entryTime = Carbon::parse($log->punch_timestamp);
    }
    if (in_array($log->machine_name, [2, 4]) && $entryTime) {
        $exitTime = Carbon::parse($log->punch_timestamp);
        $timeSpent = $exitTime->diffInSeconds($entryTime);
        $totalTime += $timeSpent;
        $entryTime = null;
    }
}
```

### 6. Attendance Approval Workflow - `attendance($request, $attendance_section_show)`

**Purpose**: Manages complex attendance/leave approval workflows

**Multi-Level Sorting System**:
1. **Priority 1**: Overdue approved requests
2. **Priority 2**: Currently active approved requests  
3. **Priority 3**: Future approved requests
4. **Priority 4**: Current pending requests (urgent)
5. **Priority 5**: Future pending requests

**Permission System**:
```php
$query->whereHas('user.userLeaveApprover', function ($subQuery) {
    $subQuery->where('approver_id', auth()->user()->user_id);
})->orWhereHas('user.userLeaveViewer', function ($q) {
    $q->where('user_id', auth()->user()->user_id);
});
```

---

## Technical Architecture

### Database Relationships
```
User
├── UserInfo (1:1)
├── ProjectMember (1:N)
├── ProjectTaskWorkLog (1:N)
├── BioMetric (1:N)
└── Attendance (1:N)

ProjectMember
├── Project (N:1)
└── ProjectTask (1:N through project)

Attendance
├── AttendanceInfo (1:N)
└── User (N:1)
```

### Performance Optimizations
- **Eager Loading**: Prevents N+1 queries
- **Selective Fields**: Reduces memory usage
- **Raw SQL**: Complex calculations in database
- **Pagination**: Limits result sets
- **Collections**: Efficient data manipulation

### Security Features
- **Authentication**: User validation
- **Authorization**: Permission-based access
- **Data Isolation**: User-specific queries
- **SQL Injection Prevention**: Eloquent ORM

---

## Business Rules Summary

### Time Tracking Rules
1. **Standard Work Day**: 8.5 hours (8 hours + 30 min lunch)
2. **Half-Day Leave**: Reduces expected hours to 4.5
3. **Time Adjustments**: Manual corrections for schedule changes
4. **Extra Work**: Additional hours beyond standard day
5. **Remote Work**: Flexible rules for remote employees

### Validation Rules
1. **Logged ≤ Biometric**: Prevents time theft
2. **Yesterday Validation**: Ensures proper hour logging
3. **Holiday Awareness**: Respects company calendar
4. **Leave Integration**: Accommodates approved leaves

### Approval Workflow
1. **Hierarchy Based**: Approvers and viewers
2. **Priority Sorting**: Urgent requests first
3. **Status Tracking**: Pending, approved, rejected
4. **Date Intelligence**: Past, current, future handling

---

## Summary

The Dashboard Controller represents a sophisticated enterprise-level time tracking system with:

- **Complex Business Logic**: Handles holidays, leaves, time adjustments
- **Performance Optimization**: Efficient queries and data processing
- **Security**: Role-based access and data protection
- **User Experience**: Personalized, timezone-aware interface
- **Compliance**: Ensures proper work hour tracking and validation

This system effectively manages employee time tracking, task management, and approval workflows while maintaining data integrity and providing excellent user experience.
