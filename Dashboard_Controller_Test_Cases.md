# Dashboard Controller Test Cases Documentation
## ERP System - Comprehensive Test Scenarios

---

## Table of Contents
1. [calculateWorkLogAndBiometricHours Test Cases](#calculateworklogandbiometrichours-test-cases)
2. [getNameSection Test Cases](#getnamesection-test-cases)
3. [getInProgressSection Test Cases](#getinprogresssection-test-cases)
4. [checkYesterdayLogError Test Cases](#checkyesterdaylogerror-test-cases)
5. [attendance Test Cases](#attendance-test-cases)
6. [punchCalculateTime Test Cases](#punchcalculatetime-test-cases)

---

## calculateWorkLogAndBiometricHours Test Cases

### **Case 1: Regular working day – Work logged but no biometric punch**
**Conditions:**
- User is not remote (`is_remote != 1`)
- Not a holiday
- Not a weekend
- Work log exists (logged hours > 0)
- No biometric punch recorded (biometric hours = 0)

**Expected Result:** 
- `is_error = 1`
- `text_message = "Logged hours not matched with biometric hour"`
- Shows discrepancy between logged and actual presence

---

### **Case 2: Work from home on holiday/weekend without biometric**
**Conditions:**
- User is not remote (`is_remote != 1`)
- It's a holiday or weekend
- Work log exists (logged hours > 0)
- No biometric punch recorded (biometric hours = 0)

**Expected Result:**
- `is_error = 1`
- Non-remote users must have biometric data even on holidays/weekends
- System flags potential policy violation

---

### **Case 3: Extra working on holiday/weekend with biometric**
**Conditions:**
- User is not remote (`is_remote != 1`)
- It's a holiday or weekend
- Work log exists
- Biometric punch is present
- Extra work attendance record exists

**Expected Result:**
- `is_error = 0` (if logged ≤ biometric + extra work)
- `biometric_hours_show = 1`
- Extra work hours added to expected calculation

---

### **Case 4: Remote worker with no hour logging requirement**
**Conditions:**
- User is remote (`is_remote = 1`)
- `need_to_log_hours != 1`
- No biometric data available
- No work log entries

**Expected Result:**
- `is_error = 0`
- `logged_hours_show = 0`
- `biometric_hours_show = 0`
- Assumes full working hours completed

---

### **Case 5: Remote worker with hour logging requirement**
**Conditions:**
- User is remote (`is_remote = 1`)
- `need_to_log_hours = 1`
- Work log exists (6 hours logged)
- No biometric data
- Expected hours = 8.5

**Expected Result:**
- `is_error = 1` (if yesterday validation)
- `logged_hours_show = 1`
- `biometric_hours_show = 0`
- Must log sufficient hours even when remote

---

### **Case 6: Half-day leave with proper logging**
**Conditions:**
- Half-day leave approved
- Expected hours reduced to 4.5
- Logged hours = 4.5
- Biometric hours = 4.5

**Expected Result:**
- `is_error = 0`
- Adjusted working hours calculation
- Leave accommodation applied

---

### **Case 7: Time adjustment with early leave**
**Conditions:**
- Time adjustment approved for 2 hours early leave
- Expected hours = 8.5 - 2 = 6.5
- Logged hours = 6.5
- Biometric hours = 6.5

**Expected Result:**
- `is_error = 0`
- Time adjustment properly applied
- Reduced expected hours

---

### **Case 8: Overtime/Extra work scenario**
**Conditions:**
- Regular working day
- Extra work approved (2 hours)
- Logged hours = 10.5
- Biometric hours = 10.5
- Expected base = 8.5

**Expected Result:**
- `is_error = 0`
- Extra work hours added to calculations
- Overtime properly tracked

---

### **Case 9: Time theft detection**
**Conditions:**
- Logged hours = 8.5
- Biometric hours = 6.0
- No extra work or adjustments

**Expected Result:**
- `is_error = 1`
- `text_message = "Logged hours not matched with biometric hour"`
- Clear time theft indication

---

### **Case 10: Biometric device error**
**Conditions:**
- Biometric records exist with error status
- Work log exists
- Regular working day

**Expected Result:**
- `biometric_error = 1`
- System acknowledges device issues
- May affect error calculation logic

---

## getNameSection Test Cases

### **Case 1: Morning greeting in user timezone**
**Conditions:**
- User timezone = "America/New_York"
- Current time in user timezone = 9:00 AM
- User full name = "John Doe"

**Expected Result:**
- `greeting = "Good Morning!"`
- `class = "morning"`
- `full_name = "John Doe"`

---

### **Case 2: Afternoon greeting with null timezone**
**Conditions:**
- User timezone = null
- App timezone = "UTC"
- Current time = 2:00 PM UTC
- User full name = "Jane Smith"

**Expected Result:**
- `greeting = "Good Afternoon!"`
- `class = "afternoon"`
- Falls back to app timezone

---

### **Case 3: Evening greeting**
**Conditions:**
- Current time = 6:30 PM
- User timezone = "Asia/Kolkata"

**Expected Result:**
- `greeting = "Good Evening!"`
- `class = "evening"`

---

### **Case 4: Night time greeting**
**Conditions:**
- Current time = 11:00 PM
- Any timezone

**Expected Result:**
- `greeting = "Good Evening!"`
- `class = "night"`
- Consistent evening greeting for night hours

---

### **Case 5: Edge case - Midnight**
**Conditions:**
- Current time = 12:00 AM (midnight)

**Expected Result:**
- `greeting = "Good Evening!"`
- `class = "night"`

---

### **Case 6: Edge case - Early morning**
**Conditions:**
- Current time = 5:59 AM

**Expected Result:**
- `greeting = "Good Evening!"`
- `class = "night"`
- Before 6 AM considered night

---

## getInProgressSection Test Cases

### **Case 1: Employee with hour logging requirement**
**Conditions:**
- User type = Employee
- `need_to_log_hours = 1`
- Has project memberships (not client type)
- Has in-progress tasks assigned

**Expected Result:**
- `in_progress_section_show = 1`
- `list` contains filtered tasks
- `count` shows zero-padded task count

---

### **Case 2: Employee without hour logging requirement**
**Conditions:**
- User type = Employee
- `need_to_log_hours = 2` (No)
- Has project memberships

**Expected Result:**
- `in_progress_section_show = 0`
- `list` is empty collection
- `count = "00"`

---

### **Case 3: Manager/Admin user**
**Conditions:**
- User type = Manager or Admin
- Has project access

**Expected Result:**
- `in_progress_section_show = 0`
- Section not shown for non-employees
- Empty task list

---

### **Case 4: Employee with no UserInfo record**
**Conditions:**
- User type = Employee
- UserInfo record is null

**Expected Result:**
- `in_progress_section_show = 0`
- Requires UserInfo for configuration
- Safe fallback to empty state

---

### **Case 5: Employee with client-only project access**
**Conditions:**
- User type = Employee
- `need_to_log_hours = 1`
- Only client-type project memberships

**Expected Result:**
- `in_progress_section_show = 1`
- `list` is empty (no internal projects)
- `count = "00"`

---

### **Case 6: Long task name truncation**
**Conditions:**
- Task name > 50 characters
- All other conditions valid

**Expected Result:**
- Task name truncated to 50 chars + "..."
- `mb_substr` used for international characters
- UI-friendly display

---

### **Case 7: Multiple in-progress tasks**
**Conditions:**
- Employee has 15 in-progress tasks
- All conditions valid

**Expected Result:**
- `count = "15"`
- All tasks in list
- Proper zero-padding for single digits

---

## checkYesterdayLogError Test Cases

### **Case 1: Regular working day with proper logging**
**Conditions:**
- Yesterday was Monday (regular working day)
- No holidays, no leaves
- Proper work hours logged
- Biometric data matches

**Expected Result:**
- `yesterday_log_error = 0`
- `is_error = 0`
- No validation errors

---

### **Case 2: Yesterday was weekend**
**Conditions:**
- Yesterday was Saturday
- No additional working day marked
- Algorithm finds previous Friday

**Expected Result:**
- Last working day = Friday
- Validation runs for Friday's data
- Weekend properly skipped

---

### **Case 3: Yesterday was holiday**
**Conditions:**
- Yesterday was national holiday
- Not marked as additional working day
- Algorithm goes back to find working day

**Expected Result:**
- Holiday skipped in calculation
- Previous working day used
- Holiday calendar respected

---

### **Case 4: Employee on leave yesterday**
**Conditions:**
- Employee had approved leave yesterday
- Leave dates properly recorded
- Algorithm finds previous working day

**Expected Result:**
- Leave day skipped
- No error for leave day
- Previous working day validated

---

### **Case 5: Multi-day leave range**
**Conditions:**
- Employee on leave from Monday to Wednesday
- Yesterday was Tuesday
- Algorithm processes date range

**Expected Result:**
- All leave dates expanded and skipped
- Finds last working day before leave
- Range processing works correctly

---

### **Case 6: Holiday moved to different date**
**Conditions:**
- Saturday holiday moved to Monday
- Monday marked as additional working day
- Yesterday was this Monday

**Expected Result:**
- Monday treated as working day
- Additional working day logic applied
- Override rules work correctly

---

### **Case 7: Half-day leave yesterday**
**Conditions:**
- Half-day leave approved for yesterday
- `is_half_day` in [2, 3] (first/second half)
- Some work expected

**Expected Result:**
- Expected hours reduced by 4
- Validation adjusted for half-day
- Partial work day handled

---

### **Case 8: Non-employee user**
**Conditions:**
- User type = Manager/Admin
- No UserInfo record

**Expected Result:**
- Default response with no errors
- `yesterday_log_error = 0`
- Early return for non-employees

---

### **Case 9: Complex scenario - Holiday on leave**
**Conditions:**
- Yesterday was both holiday and employee leave
- Additional working day also marked
- Multiple conditions overlap

**Expected Result:**
- Proper precedence handling
- Additional working day may override
- Complex logic resolved correctly

---

## attendance Test Cases

### **Case 1: User with approval permissions**
**Conditions:**
- User has `attendance.index` or `attendance.show` permission
- Is approver for some employees
- Pending requests exist

**Expected Result:**
- `access_attendance = true`
- `data` contains filtered requests
- Proper permission-based filtering

---

### **Case 2: User without permissions**
**Conditions:**
- User lacks attendance permissions
- Not an approver or viewer

**Expected Result:**
- `access_attendance = false`
- `data` is empty array
- Early return for unauthorized access

---

### **Case 3: Urgent today requests prioritization**
**Conditions:**
- Multiple requests: past, today, future
- Today's requests need immediate attention
- Complex sorting applied

**Expected Result:**
- Today's requests appear first
- Proper priority ordering
- Urgent items highlighted

---

### **Case 4: Leave approver hierarchy**
**Conditions:**
- User is approver for specific employees
- Requests from those employees exist
- Hierarchy properly configured

**Expected Result:**
- Only relevant requests shown
- Approver relationship respected
- Proper data filtering

---

### **Case 5: Leave viewer access**
**Conditions:**
- User is viewer (not approver)
- Has limited access to requests
- Can see but may not approve

**Expected Result:**
- Requests visible based on viewer role
- Access level properly handled
- Role-based data access

---

### **Case 6: Mixed request types and statuses**
**Conditions:**
- Pending, approved, cancelled requests
- Different leave types (full day, half day, etc.)
- Complex sorting requirements

**Expected Result:**
- Proper status-based sorting
- Type-aware prioritization
- Consistent ordering logic

---

### **Case 7: Large dataset with pagination**
**Conditions:**
- More than 100 requests
- Pagination limit applied
- Performance considerations

**Expected Result:**
- Results limited to 100 items
- Pagination properly applied
- Performance maintained

---

## punchCalculateTime Test Cases

### **Case 1: Normal punch in/out cycle**
**Conditions:**
- Punch in at 9:00 AM (machine code 1)
- Punch out at 6:00 PM (machine code 2)
- Single work session

**Expected Result:**
- Total time = 9 hours
- Formatted as "9h 0m"
- Correct time calculation

---

### **Case 2: Multiple punch cycles in a day**
**Conditions:**
- Punch in 9:00 AM, out 1:00 PM (4 hours)
- Punch in 2:00 PM, out 6:00 PM (4 hours)
- Total = 8 hours

**Expected Result:**
- Both sessions calculated
- Total time = 8 hours
- Multiple cycles handled

---

### **Case 3: Currently punched in (active session)**
**Conditions:**
- Last punch was entry (code 1 or 3)
- Timestamp is today
- No corresponding exit

**Expected Result:**
- Time calculated until current moment
- Active session included
- Real-time calculation

---

### **Case 4: Invalid/missing timestamps**
**Conditions:**
- Some records have null timestamps
- Some records have blank values
- Mixed valid and invalid data

**Expected Result:**
- Invalid records skipped gracefully
- Valid records processed
- No errors from bad data

---

### **Case 5: Unpaired punch records**
**Conditions:**
- Punch in without corresponding punch out
- Multiple consecutive punch ins
- Data integrity issues

**Expected Result:**
- Unpaired entries handled safely
- Entry time reset on new entry
- No calculation errors

---

### **Case 6: Different machine codes**
**Conditions:**
- Machine codes 1,3 for entry
- Machine codes 2,4 for exit
- Mixed machine types

**Expected Result:**
- All valid codes recognized
- Proper entry/exit pairing
- Machine type flexibility

---

### **Case 7: Cross-day punch scenario**
**Conditions:**
- Punch in late evening
- Punch out next morning
- Spans midnight

**Expected Result:**
- Time calculation across days
- Proper duration computed
- Date boundary handled

---

### **Case 8: No punch records**
**Conditions:**
- Empty biometric data
- No punch records for date

**Expected Result:**
- Total time = 0
- Formatted as "0h 0m"
- Graceful handling of no data

---

### **Case 9: Biometric device errors**
**Conditions:**
- Some records marked with error status
- Mixed error and valid records
- Device malfunction scenarios

**Expected Result:**
- Error records may be skipped
- Valid records still processed
- Error status tracked separately

---

## Edge Cases and Error Scenarios

### **System-Level Edge Cases**

#### **Case 1: Database connection issues**
**Conditions:**
- Database temporarily unavailable
- Query timeouts
- Connection failures

**Expected Result:**
- Graceful error handling
- Default safe values returned
- System remains stable

---

#### **Case 2: Null/missing user data**
**Conditions:**
- User record incomplete
- UserInfo missing critical fields
- Data integrity issues

**Expected Result:**
- Safe defaults applied
- No fatal errors
- Fallback values used

---

#### **Case 3: Timezone conversion errors**
**Conditions:**
- Invalid timezone strings
- Timezone data corruption
- DST transition issues

**Expected Result:**
- Fallback to system timezone
- Error logging
- Continued operation

---

#### **Case 4: Large dataset performance**
**Conditions:**
- Thousands of punch records
- Complex date range queries
- Memory constraints

**Expected Result:**
- Pagination applied
- Query optimization
- Memory management

---

### **Business Logic Edge Cases**

#### **Case 1: Conflicting attendance records**
**Conditions:**
- Leave and extra work on same day
- Multiple time adjustments
- Overlapping records

**Expected Result:**
- Precedence rules applied
- Conflicts resolved logically
- Consistent calculations

---

#### **Case 2: Retroactive data changes**
**Conditions:**
- Holiday calendar updated retroactively
- Leave approvals changed
- Historical data modifications

**Expected Result:**
- Calculations reflect current data
- Historical accuracy maintained
- Change impact handled

---

#### **Case 3: International operations**
**Conditions:**
- Multiple country holidays
- Different working week patterns
- Cultural considerations

**Expected Result:**
- Localized holiday handling
- Flexible working patterns
- Cultural awareness

---

## Test Data Requirements

### **User Test Data**
- Employee users with different configurations
- Manager/Admin users
- Remote and office-based users
- Users with different timezone settings

### **Time Data**
- Various date ranges (today, yesterday, week, month)
- Holiday periods and weekends
- Leave periods and time adjustments
- Biometric punch records

### **Configuration Data**
- Working hours configurations
- Holiday calendars
- Leave policies
- Project and task assignments

### **Permission Data**
- Role-based permissions
- Approval hierarchies
- Viewer access rights
- Security configurations

---

## Validation Checklist

### **Function Validation**
- [ ] All functions handle null inputs gracefully
- [ ] Error conditions return appropriate messages
- [ ] Performance remains acceptable with large datasets
- [ ] Security permissions properly enforced

### **Business Logic Validation**
- [ ] Time calculations are mathematically correct
- [ ] Holiday and leave logic works as expected
- [ ] Approval workflows function properly
- [ ] Edge cases handled appropriately

### **Integration Validation**
- [ ] Database queries perform efficiently
- [ ] API responses are properly formatted
- [ ] Frontend integration works smoothly
- [ ] Error handling is user-friendly

This comprehensive test case documentation covers all major functions, edge cases, and business scenarios for the Dashboard Controller system.
