# Dashboard Controller - Validation Test Cases

## Test Case Format

Each test case follows this structure:
- **Case Description**: What scenario we're testing
- **Conditions**: Specific system states and user attributes
- **Expected Result**: What the Dashboard should show
- **Business Impact**: Why this validation matters
- **Test Steps**: How to reproduce this scenario

---

## Working Hours Validation Test Cases

### **Case 1: Regular working day – Work logged but no biometric punch**

**Description**: Office employee logs work hours but punch-in/out system has no record

**Conditions:**
- User is not remote (is_remote = NO)
- Not a holiday (date not in holiday calendar)
- Not a weekend (Monday-Friday)
- Work log exists (time_spent > 0 in project_task_work_logs)
- No biometric punch recorded (no records in biometrics table)
- User requires hour logging (need_to_log_hours = YES)

**Expected Result**: Error
- Status: ❌ Red error indicator
- Message: "Missing punch-in/punch-out records"
- Biometric Hours: "0h 0m"
- Logged Hours: Shows actual logged time (e.g., "8h 30m")
- Error Flag: is_error = 1
- Biometric Error: biometric_error = 1

**Business Impact**: 
- Prevents payroll processing without attendance verification
- Ensures office employees are physically present when claiming work hours
- Maintains compliance with labor laws requiring presence tracking

**Test Steps:**
1. Set up office employee (is_remote = NO, need_to_log_hours = YES)
2. Add work log entries for a regular weekday (e.g., 8 hours)
3. Ensure no biometric records exist for that date
4. Verify date is not holiday or weekend
5. Access dashboard and check "Today's Working Hours" section
6. Confirm error status and appropriate messaging

**Technical Details:**
- `$biometric_hours_show = 1` (should show for office workers)
- `$logged_hours_show = 1` (should show for hour-logging employees)  
- `$biometricHoursInMinutes = 0` (no punch records)
- `$loggedHoursInMinutes > 0` (work was logged)
- Error triggered because logged > biometric when biometric = 0

---

### **Case 2: Work from home on holiday/weekend without biometric**

**Description**: Office employee (not designated remote) logs work on non-working day without punch records

**Conditions:**
- User is not remote (is_remote = NO) 
- It's a holiday (date exists in holiday_etcs table, type != 3) OR weekend (Saturday/Sunday)
- Work log exists (entries in project_task_work_logs)
- No biometric punch recorded
- User requires hour logging (need_to_log_hours = YES)
- No approved extra work request exists

**Expected Result**: Error
- Status: ❌ Red error indicator
- Message: "Unauthorized work on non-working day without approval"
- Biometric Hours: "0h 0m"
- Logged Hours: Shows logged time
- Error Flag: is_error = 1
- Additional validation: Should check for extra work approval

**Business Impact**:
- Prevents unauthorized overtime without proper approval
- Ensures holiday work is pre-approved and properly compensated
- Maintains work-life balance policies
- Controls labor costs on non-standard days

**Test Steps:**
1. Set up office employee (is_remote = NO)
2. Select a holiday date or weekend day
3. Add work log entries for that date
4. Ensure no biometric punch records exist
5. Verify no extra work approval exists for that date
6. Access dashboard for that date period
7. Confirm error status and policy violation message

**Technical Details:**
- Holiday check: `HolidayEtc::whereNot('type', 3)->whereNull('deleted_at')`
- Weekend check: `$date->dayOfWeek in [Carbon::SATURDAY, Carbon::SUNDAY]`
- Extra work check: `Attendance::TYPE_EXTRA_WORKING_*` with approved status
- Error because work logged on non-working day without authorization

---

### **Case 3: Extra working on holiday/weekend with biometric**

**Description**: Office employee works on holiday/weekend with punch records but without proper extra work approval

**Conditions:**
- User is not remote (is_remote = NO)
- It's a holiday OR weekend
- Work log exists  
- Biometric punch is present (valid punch-in/out records)
- User requires hour logging (need_to_log_hours = YES)
- No approved extra work request exists for that date

**Expected Result**: Error
- Status: ❌ Red error indicator  
- Message: "Holiday/weekend work requires pre-approval"
- Biometric Hours: Shows calculated punch time (e.g., "8h 45m")
- Logged Hours: Shows logged time (e.g., "8h 30m")
- Error Flag: is_error = 1
- Additional Note: "Submit extra work request for approval"

**Business Impact**:
- Ensures all holiday/weekend work is properly authorized
- Controls overtime costs and prevents surprise labor expenses
- Maintains compliance with union contracts and labor agreements
- Protects employee rights regarding mandatory rest days

**Test Steps:**
1. Set up office employee (is_remote = NO)  
2. Select holiday or weekend date
3. Add biometric punch records (in/out times)
4. Add corresponding work log entries
5. Ensure no extra work approval exists
6. Access dashboard and verify error status
7. Confirm guidance about approval requirement

**Technical Details:**
- Biometric calculation: `punchCalculateTime()` processes punch records
- Holiday/weekend detection: Date validation against holiday calendar and weekdays
- Extra work approval check: `Attendance::whereIn('type', [TYPE_EXTRA_WORKING_WFH, TYPE_EXTRA_WORKING])`
- Error because holiday work exists without proper approval workflow

---

### **Case 4: Remote employee working on regular day**

**Description**: Remote employee logs work on regular business day (should pass validation)

**Conditions:**
- User is remote (is_remote = YES)
- Regular working day (Monday-Friday, not holiday)
- Work log exists
- No biometric punch expected/required
- User requires hour logging (need_to_log_hours = YES)
- Work hours within expected range

**Expected Result**: Success
- Status: ✅ Green success indicator
- Message: "Remote work hours validated"
- Biometric Hours: "Not Applicable" or hidden section
- Logged Hours: Shows logged time (e.g., "8h 15m")
- Error Flag: is_error = 0
- Remote work status clearly indicated

**Business Impact**:
- Supports flexible remote work policies
- Validates productivity without requiring physical presence
- Maintains accurate payroll for distributed workforce
- Ensures remote workers meet hour commitments

**Test Steps:**
1. Set up remote employee (is_remote = YES)
2. Select regular business day
3. Add work log entries within normal range
4. Verify no biometric records required
5. Access dashboard and confirm success status
6. Verify appropriate messaging for remote work

---

### **Case 5: Half-day leave with partial work**

**Description**: Employee on approved half-day leave logs appropriate partial hours

**Conditions:**
- User is not remote (is_remote = NO)
- Regular working day
- Approved half-day leave exists (is_half_day = 2 or 3)
- Work log exists for remaining half-day
- Biometric punch shows partial presence
- Expected hours reduced by 4 hours for half-day

**Expected Result**: Success
- Status: ✅ Green success indicator
- Message: "Half-day schedule validated"
- Expected hours automatically adjusted (8.5h → 4.5h)
- Logged and biometric hours match reduced expectation
- Leave status displayed prominently
- Error Flag: is_error = 0

**Business Impact**:
- Accurately handles partial work days and leave
- Prevents errors when employees have approved reduced schedules
- Ensures proper payroll calculation for partial days
- Maintains leave balance accuracy

**Test Steps:**
1. Set up employee with approved half-day leave
2. Add work logs for half day (e.g., 4 hours)
3. Add biometric records showing partial presence
4. Verify system reduces expected hours by 4
5. Access dashboard and confirm success with adjusted expectations
6. Verify leave information is displayed

---

### **Case 6: Overtime with pre-approval**

**Description**: Employee works overtime hours with proper advance approval

**Conditions:**
- User is not remote (is_remote = NO)
- Regular working day
- Work log shows overtime hours (>8.5 standard)
- Biometric punch shows extended presence
- Approved extra work request exists (status = APPROVED)
- Overtime hours match approved request

**Expected Result**: Success  
- Status: ✅ Green success indicator
- Message: "Approved overtime validated"
- Expected hours increased by approved overtime amount
- Logged and biometric hours include overtime
- Approval status displayed
- Error Flag: is_error = 0
- Overtime clearly identified and approved

**Business Impact**:
- Enables controlled overtime when business needs require
- Prevents unauthorized overtime costs
- Ensures proper compensation for extra work
- Maintains audit trail for labor cost management

**Test Steps:**
1. Set up employee with approved overtime request
2. Add work logs including overtime hours
3. Add biometric records showing extended presence
4. Verify overtime approval exists and matches hours
5. Access dashboard and confirm validation passes
6. Verify overtime status and approval are displayed

---

### **Case 7: Manager without detailed hour tracking**

**Description**: Manager-level employee who doesn't log detailed task hours

**Conditions:**
- User is manager/supervisor level
- User does not require hour logging (need_to_log_hours = NO)
- Regular working day
- Biometric punch exists (if office-based)
- No work log entries expected

**Expected Result**: Success (Limited Display)
- Status: ℹ️ Info status or no specific validation
- Message: "Management schedule - detailed logging not required"
- Biometric Hours: Shown if office-based
- Logged Hours: Section hidden or shows "Not Required"
- No error flags
- Different validation rules applied

**Business Impact**:
- Accommodates different job roles and responsibilities
- Reduces administrative burden for management positions
- Maintains attendance tracking without micromanagement
- Supports flexible work arrangements for senior staff

**Test Steps:**
1. Set up manager-level user (need_to_log_hours = NO)
2. Add biometric punch records if office-based
3. Verify no work log entries are required
4. Access dashboard and confirm appropriate display
5. Verify no task section appears
6. Confirm biometric tracking still applies if office-based

---

### **Case 8: Missing yesterday validation on Monday**

**Description**: System validates Friday's work when employee logs in on Monday

**Conditions:**
- Current day is Monday
- Previous work day was Friday (skipping weekend)
- Friday had work logs and biometric data
- Saturday/Sunday correctly ignored as non-work days
- All Friday data validates correctly

**Expected Result**: Success
- Status: ✅ Green validation for Friday
- Message: "Previous work day (Friday) validated"
- Date shown: Friday's date, not Sunday
- Yesterday section shows Friday's data
- Weekend days properly skipped
- Error Flag: is_error = 0

**Business Impact**:
- Intelligent working day calculation prevents false errors
- Maintains validation continuity across weekends
- Reduces Monday morning error alerts for valid scenarios
- Supports standard business week operations

**Test Steps:**
1. Set up scenario where today is Monday
2. Ensure Friday had complete work and biometric data
3. Verify Saturday/Sunday have no work expectations
4. Access dashboard on Monday
5. Confirm Friday validation, not weekend error
6. Verify proper date recognition and skipping

---

### **Case 9: New employee first week**

**Description**: Brand new employee during first week of employment

**Conditions:**
- Employee start date is current week
- Limited historical data available
- May not have all systems access yet
- First few days of work logs
- Learning curve and system setup period

**Expected Result**: Success (Graceful Handling)
- Status: ℹ️ Info status or gentle validation
- Message: "New employee - building work history"
- Reduced validation strictness during orientation period
- Helpful onboarding messages
- No harsh error flags for expected gaps
- Guidance on system usage

**Business Impact**:
- Smooth onboarding experience for new hires
- Prevents discouraging error messages during learning phase
- Maintains system integrity while accommodating new users
- Supports HR onboarding processes

**Test Steps:**
1. Set up employee with recent start date
2. Add limited work history (first few days)
3. May have some system access delays
4. Access dashboard and verify gentle treatment
5. Confirm onboarding messages and guidance
6. Verify no punitive error handling

---

### **Case 10: Holiday converted to working day**

**Description**: Company holiday marked as additional working day for specific projects

**Conditions:**
- Date is normally company holiday
- Holiday marked as additional working day (type = 3)
- Employee scheduled to work
- Work logs and biometric data present
- Special working day properly configured

**Expected Result**: Success
- Status: ✅ Green validation
- Message: "Special working day validated"
- Holiday override clearly indicated
- Normal working day validation applied
- Override authorization displayed
- Error Flag: is_error = 0

**Business Impact**:
- Supports flexible business operations during critical periods
- Accommodates client deadlines and project requirements
- Maintains proper compensation for holiday work
- Provides clear audit trail for special arrangements

**Test Steps:**
1. Set up holiday date with working day override
2. Add normal work logs and biometric data
3. Verify holiday override is properly configured
4. Access dashboard and confirm normal validation
5. Verify special working day status is displayed
6. Confirm proper handling of holiday pay implications

---

### **Case 11: Biometric system error with manual override**

**Description**: Biometric system malfunction with HR manual time adjustment

**Conditions:**
- Biometric system offline or malfunctioning
- Work logs exist showing work was performed
- HR manual time adjustment entered
- Manual adjustment matches work logs
- System error properly documented

**Expected Result**: Success (With Override)
- Status: ⚠️ Warning status (manual intervention)
- Message: "Manual time adjustment applied"
- Biometric Hours: Shows manual adjustment
- Logged Hours: Shows work log data
- Manual override clearly indicated
- Audit trail maintained

**Business Impact**:
- Maintains payroll accuracy despite technical failures
- Provides fallback process for system outages
- Ensures employees aren't penalized for technical issues
- Maintains operational continuity

**Test Steps:**
1. Simulate biometric system failure
2. Add work logs for affected day
3. Create HR manual time adjustment
4. Verify manual override is properly applied
5. Access dashboard and confirm override handling
6. Verify audit trail and documentation

---

### **Case 12: Partial day medical appointment**

**Description**: Employee leaves early for medical appointment with time adjustment

**Conditions:**
- Regular working day
- Employee left early for medical appointment
- Time adjustment request submitted and approved
- Reduced biometric hours due to early departure
- Work logs match actual time present

**Expected Result**: Success
- Status: ✅ Green validation
- Message: "Medical leave time adjustment applied"
- Expected hours reduced by approved adjustment
- Biometric and logged hours align with adjustment
- Medical leave status indicated
- Error Flag: is_error = 0

**Business Impact**:
- Accommodates medical appointments and health needs
- Ensures accurate payroll for partial attendance
- Maintains employee wellbeing support
- Provides proper documentation for adjustments

**Test Steps:**
1. Set up employee with approved medical time adjustment
2. Add work logs for partial day
3. Add biometric records showing early departure
4. Verify time adjustment approval exists
5. Access dashboard and confirm proper validation
6. Verify medical accommodation is properly handled

---

## Quick Reference Error Matrix

| Condition | Remote | Holiday/Weekend | Work Log | Biometric | Result | Primary Reason |
|-----------|--------|-----------------|----------|-----------|--------|----------------|
| Regular Day | NO | NO | YES | NO | ❌ Error | Missing biometric for office worker |
| Holiday | NO | YES | YES | NO | ❌ Error | Unauthorized work without approval |
| Holiday | NO | YES | YES | YES | ❌ Error | Holiday work without pre-approval |
| Regular Day | YES | NO | YES | NO | ✅ Success | Remote work doesn't require biometric |
| Regular Day | NO | NO | YES | YES | ✅ Success | Normal office work properly tracked |
| Half Leave | NO | NO | PARTIAL | PARTIAL | ✅ Success | Leave adjustment applied correctly |

## Testing Checklist

### Pre-Test Setup
□ User roles and permissions configured correctly  
□ Holiday calendar up to date  
□ Leave approvals in system  
□ Biometric system status verified  
□ Work log test data prepared  

### During Testing  
□ Test each condition combination systematically  
□ Verify error messages are clear and actionable  
□ Confirm business logic matches company policies  
□ Check audit trails are properly maintained  
□ Validate user experience is intuitive  

### Post-Test Validation
□ Document any discrepancies found  
□ Verify fixes don't break other scenarios  
□ Confirm performance remains acceptable  
□ Update documentation with any changes  
□ Train users on proper procedures  

---

This comprehensive test case format ensures thorough validation of all Dashboard Controller scenarios while maintaining clear business context and actionable results.
