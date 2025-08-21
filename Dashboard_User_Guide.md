# ERP Dashboard - User Guide

## What is the Dashboard?

The ERP Dashboard is your **central control center** that shows you everything important about your workday at a glance. Think of it like the homepage of your work life - it displays your tasks, working hours, attendance status, and any requests that need your attention.

---

## Dashboard Sections Overview

When you log into the system, your dashboard shows several sections. Here's what each one does:

### 🌅 **Welcome Section**
**What you see:** A personalized greeting like "Good Morning, John!"

**What it does:**
- Shows your name and a time-appropriate greeting
- Changes throughout the day (Good Morning → Good Afternoon → Good Evening)
- Uses your timezone to show the correct greeting

**Why it's useful:** Makes the system feel personal and welcoming.

---

### 📋 **My Active Tasks Section**
**What you see:** A list of tasks you're currently working on

**What it does:**
- Shows only tasks assigned to you that are "In Progress"
- Displays task names (shortened if too long)
- Shows how many active tasks you have
- Only appears if you're required to log work hours

**Why it's useful:** Quick overview of what you should be working on today.

**Example:**
```
Active Tasks (03)
• Fix login bug for mobile app...
• Update user dashboard design
• Review new feature requirements
```

---

### ⏰ **Today's Working Hours**
**What you see:** Your logged time vs your actual presence time for today

**What it does:**
- **Logged Hours:** Time you recorded working on tasks (if required)
- **Biometric Hours:** Time recorded by punch-in/punch-out system (if applicable)
- **Status Indicator:** Shows if your recorded time matches expectations
- **Error Alerts:** Warns if there are discrepancies

**Why it's useful:** Helps ensure you're tracking your time accurately.

**Example:**
```
Today's Hours
Logged: 6h 30m | Biometric: 7h 15m
Status: ✅ Time tracking is accurate
```

---

### 📅 **Yesterday's Working Hours**
**What you see:** Analysis of your previous working day

**What it does:**
- Checks if you completed proper time tracking for yesterday
- **Smart Date Calculation:** Automatically skips weekends, holidays, and your leave days
- **Error Detection:** Flags if you forgot to log hours or if times don't match
- **Holiday Awareness:** Knows company holidays and doesn't count them as work days

**Why it's useful:** Ensures you don't miss logging time for completed work days.

**Example:**
```
Yesterday (Aug 20, 2024)
Logged: 8h 0m | Biometric: 8h 15m
Status: ⚠️ Please review your time entries
```

---

### 📊 **Weekly Summary**
**What you see:** Current week's total working hours

**What it does:**
- Adds up all your logged and biometric hours for the week (Monday-Sunday)
- Shows progress toward your weekly hour goals
- Includes any approved overtime or extra work

**Why it's useful:** Track weekly progress and ensure you're meeting hour requirements.

---

### 📈 **Monthly Overview**
**What you see:** Current month's total working hours

**What it does:**
- Shows month-to-date totals for logged and biometric hours
- Useful for monthly performance reviews
- Includes all approved time adjustments and extra work

**Why it's useful:** Long-term view of your work patterns and productivity.

---

### 📝 **Attendance Requests**
**What you see:** Leave requests and time adjustments that need your attention

**What it does:**
- **If you're a Manager/Approver:** Shows requests from your team that need approval
- **If you're an HR Viewer:** Shows attendance requests you can view
- **Smart Prioritization:** Most urgent requests appear first
- **Request Types:** Leave requests, time adjustments, extra work notifications

**Why it's useful:** Manage team attendance efficiently without missing important requests.

**Example:**
```
Pending Requests (5)
🔥 Sarah Johnson - Sick Leave (Today) - URGENT
📅 Mike Chen - Vacation Request (Next Week)
⏰ Lisa Wong - Time Adjustment (Yesterday)
```

---

## How the System is Smart

### 🧠 **Intelligent Working Day Detection**
The system is smart about what counts as a working day:

- **Skips Weekends:** Doesn't expect work on Saturday/Sunday (unless marked as working days)
- **Knows Holidays:** Automatically excludes company holidays from work day calculations
- **Respects Your Leave:** Doesn't count days you're on approved vacation or sick leave
- **Handles Special Cases:** Some holidays might be working days, and the system knows this

### 🏠 **Different Rules for Different Workers**
The system understands that not everyone works the same way:

**Office Workers:**
- Must punch in/out using biometric systems
- System compares logged task time with punch-in/out time
- Shows both types of time tracking

**Remote Workers:**
- Don't need to punch in/out
- May or may not need to log task hours (depends on role)
- System adjusts expectations accordingly

**Managers/Seniors:**
- May not need to log detailed task hours
- Still get overview of team activities
- Can approve/view team attendance requests

### ⚖️ **Smart Time Validation**
The system helps catch common time tracking mistakes:

**Over-logging Detection:**
- If you log more task hours than you were actually present
- Helps prevent accidental double-counting

**Under-logging Detection:**
- If you didn't log enough hours for a full work day
- Reminds you to record all your work

**Special Situations:**
- Half-day leaves automatically adjust expected hours
- Extra work (overtime) is properly counted
- Time adjustments from HR are automatically applied

---

## What Each Status Means

### ✅ **Green/Success Status**
- Everything looks good
- Your time tracking is complete and accurate
- No action needed

### ⚠️ **Yellow/Warning Status**
- Minor issue that should be reviewed
- Might need to add missing time logs
- Usually easy to fix

### ❌ **Red/Error Status**
- Important discrepancy found
- Immediate attention needed
- May affect payroll or attendance records

### ℹ️ **Blue/Info Status**
- Informational message
- No action required
- Just keeping you informed

---

## Why This Matters for Your Organization

### 📊 **Accurate Payroll**
- Ensures everyone is paid correctly for hours worked
- Prevents under-payment or over-payment issues
- Maintains fair compensation across the organization

### 📈 **Project Management**
- Helps track how much time projects actually take
- Improves future project planning and budgeting
- Identifies bottlenecks and efficiency opportunities

### 🏆 **Performance Tracking**
- Provides data for performance reviews
- Shows work patterns and productivity trends
- Helps identify training needs or workload issues

### ⚖️ **Compliance**
- Meets labor law requirements for time tracking
- Provides audit trails for regulatory compliance
- Ensures proper overtime calculation and documentation

### 👥 **Team Management**
- Managers get clear view of team activities
- Easy approval process for leave requests
- Reduces administrative overhead

---

## Common Scenarios Explained

### 🤧 **"I was sick yesterday but forgot to apply for leave"**
**What happens:** Dashboard shows a red error for yesterday because it expected you to work but you didn't log any hours.

**How to fix:** Apply for sick leave for yesterday through the HR system. Once approved, the error will automatically disappear.

---

### 🏖️ **"I'm going on vacation next week"**
**What happens:** You apply for vacation leave through the system.

**Dashboard impact:** Those vacation days won't show up as errors in your dashboard. The system knows you're not supposed to work those days.

---

### 💻 **"I worked from home and did extra hours"**
**What happens:** You log your task hours as normal, and apply for "Extra Work" to record the overtime.

**Dashboard impact:** The extra hours are counted toward your total, so you won't get under-logging errors.

---

### 🏢 **"I forgot to punch out yesterday"**
**What happens:** Biometric system might show you as still "punched in" or have incomplete data.

**Dashboard impact:** May show timing discrepancies. Contact your supervisor or HR to correct the punch record.

---

### 👔 **"I'm a manager - what requests need my attention?"**
**What you see:** The attendance section shows requests from your team members, with urgent ones listed first.

**Priority order:**
1. Requests for today (most urgent)
2. Overdue requests
3. Future requests
4. Cancelled/rejected requests

---

## Tips for Best Results

### ✅ **Daily Habits**
- Check your dashboard first thing each morning
- Review any error messages and address them promptly
- Log your work hours consistently if required
- Punch in/out accurately if using biometric system

### 📅 **Weekly Habits**
- Review your weekly totals every Friday
- Apply for any needed time adjustments before the week ends
- Plan ahead for upcoming leave requests

### 📋 **For Managers**
- Check attendance requests daily
- Approve or deny requests promptly
- Review team time patterns for unusual activities

### 🎯 **Best Practices**
- Apply for leave in advance when possible
- Keep detailed notes about your work for accurate time logging
- Report system issues to IT immediately
- Ask HR if you're unsure about any policies

---

## Getting Help

### 🐛 **Technical Issues**
- System not loading correctly
- Error messages that don't make sense
- Punch in/out problems

**Contact:** IT Support Team

### 📋 **Policy Questions**
- How many hours should I log?
- What counts as working time?
- Leave policy questions

**Contact:** Human Resources

### 👥 **Approval Issues**
- Requests stuck in pending status
- Need urgent approval
- Disagreement about time records

**Contact:** Your Direct Manager

---

## Summary

The ERP Dashboard is designed to make time tracking and attendance management as simple and accurate as possible. It automatically handles complex business rules, so you don't have to remember every policy detail.

**Key Benefits:**
- **Saves Time:** Automated calculations and smart error detection
- **Reduces Mistakes:** Built-in validation catches common errors
- **Improves Fairness:** Consistent rules applied to everyone
- **Simplifies Management:** Easy approval processes and clear priority systems

**Remember:** The system is here to help you, not to catch you making mistakes. When it shows warnings or errors, it's usually trying to help you avoid problems with payroll or attendance records.

**Questions?** Don't hesitate to ask your manager, HR team, or IT support for help understanding any part of the system.
