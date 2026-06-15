# Admin - Repayment Plan Setup Regression Test Cases

## Document Information
- **Product**: TMP Admin - Repayment Plan Management
- **Test Environment**: UAT or INT (selected at run time)
- **Last Updated**: 2026-03-13
- **Test Type**: Regression Testing - Admin Repayment Plan Setup
- **User Role**: Admin / Super Admin

---

## Test Data Configuration

### Admin Credentials
See `config/userdetails.md` (Admin user).

### Loan Parameters (Test Application)
- **Application ID**: 13243
- **Loan ID**: 3058
- **Borrower**: Accept Test
- **Loan Amount**: £999.00
- **Term**: 6 weeks
- **Daily Rate**: 0.8%
- **Installments**: 2

### Repayment Plan Parameters
- **Amount**: £999.00
- **Installment Amount**: £200.00
- **Frequency**: Monthly
- **Placement Type**: Last Weekday
- **Start Date**: 30/03/2026

> All URLs in step tables are relative paths. The runner prefixes them with the
> selected environment's base URL (UAT or INT).

---

## 1. Admin can setup repayment plan for a specific loan

### TC-RP-001: Admin Login
**Objective**: Verify admin can successfully login to TMP admin panel
**Priority**: Critical
**Preconditions**: Valid admin credentials

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/login` | Login page displayed with Email, Password fields and Login button. **FAIL** if 500 ISE error is displayed |
| 2 | Enter the admin email from userdetails | Email field populated. **FAIL** if 500 ISE error is displayed |
| 3 | Enter the admin password from userdetails | Password field populated (masked). **FAIL** if 500 ISE error is displayed |
| 4 | Click "Login" button | Redirect to admin dashboard `/admin/dashboard`. **FAIL** if 500 ISE error is displayed |
| 5 | Verify admin dashboard loaded | "The Money Platform" heading displayed, Today's Payments section visible. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Admin logged in and dashboard accessible
**Fail Criteria**: Login fails, dashboard not accessible, or 500 ISE error on any step

---

### TC-RP-002: Admin can setup repayment plan for a specific loan
**Objective**: Verify admin can setup repayment plan for a specific loan
**Priority**: High
**Preconditions**: Admin logged in (from TC-RP-001)

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/admin/loans/list/1/started` | Loan page displayed with list of started loans. **FAIL** if 500 ISE error is displayed |
| 2 | Select any one loan from the list and click "Details" | Loan details page displayed with loan summary. **FAIL** if 500 ISE error is displayed |
| 3 | Verify application state | State badge displayed (e.g., "loaned"). **FAIL** if 500 ISE error is displayed |
| 4 | Note the "Amount to be repaid £XYZ" from the loan summary panel | Loan summary with amount displayed. **FAIL** if 500 ISE error is displayed |
| 5 | Click "Repayment Plan" button | Repayment plan setup form displayed. **FAIL** if 500 ISE error is displayed |
| 6 | Enter the Amount as £XYZ (from step 4) | Amount field populated with loan amount. **FAIL** if 500 ISE error is displayed |
| 7 | Enter an installment amount less than £XYZ | Installment amount field populated. **FAIL** if 500 ISE error is displayed |
| 8 | Select frequency "Monthly" | Monthly frequency selected. **FAIL** if 500 ISE error is displayed |
| 9 | Select placement type "Last Weekday" | Last Weekday placement type selected. **FAIL** if 500 ISE error is displayed |
| 10 | Verify application state | State badge displayed (e.g., "planned"). **FAIL** if 500 ISE error is displayed |
| 11 | Click "Accept Test" borrower name link, then click "Impersonate this user" | Redirected to borrower dashboard as impersonated user. **FAIL** if 500 ISE error is displayed |
| 12 | Verify "You are currently on a Repayment Plan." text | Repayment plan confirmation text visible. **FAIL** if 500 ISE error is displayed |
| 13 | Navigate to `/logout` | User logged out and redirected to homepage. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Repayment plan setup by admin works without errors
**Fail Criteria**: Page 404/500 ISE error on any step, or incorrect data displayed

---

## 2. Borrower can setup repayment plan for a specific loan

### TC-RP-003: Admin Login (for impersonation)
**Objective**: Admin login as a precondition to impersonate a borrower
**Priority**: Critical

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/login` | Login page displayed. **FAIL** if 500 ISE error is displayed |
| 2 | Enter the admin email from userdetails | Email field populated. **FAIL** if 500 ISE error is displayed |
| 3 | Enter the admin password from userdetails | Password field populated. **FAIL** if 500 ISE error is displayed |
| 4 | Click "Login" button | Redirect to `/admin/dashboard`. **FAIL** if 500 ISE error is displayed |
| 5 | Verify admin dashboard loaded | "The Money Platform" heading displayed. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Admin logged in
**Fail Criteria**: Login fails or 500 ISE error

---

### TC-RP-004: Borrower can setup repayment plan for a specific loan
**Objective**: Verify borrower can setup repayment plan for a specific loan
**Priority**: High
**Preconditions**: Admin logged in (TC-RP-003)

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/admin/loans/list/1/started` | Loan page displayed with started loans. **FAIL** if 500 ISE error is displayed |
| 2 | Select a loan and click "Details" | Loan details page displayed. **FAIL** if 500 ISE error is displayed |
| 3 | Verify application state | State badge displayed (e.g., "loaned"). **FAIL** if 500 ISE error is displayed |
| 4 | Click "Accept Test" borrower name link | User details page displayed. **FAIL** if 500 ISE error is displayed |
| 5 | Click "Toggle User Flag" | User flag toggled successfully. **FAIL** if 500 ISE error is displayed |
| 6 | Click "Impersonate this user" | Redirected to borrower dashboard. **FAIL** if 500 ISE error is displayed |
| 7 | Click "Help with my loan" | Help options displayed. **FAIL** if button missing or 500 ISE |
| 8 | Click "Enter a repayment plan" | Repayment plan setup form displayed. **FAIL** if button missing or 500 ISE |
| 9 | Enter installment amount less than total balance | Amount field populated. **FAIL** if 500 ISE error is displayed |
| 10 | Select frequency "Monthly" | Monthly frequency selected. **FAIL** if 500 ISE error is displayed |
| 11 | Select placement type "Last Weekday" | Last Weekday placement type selected. **FAIL** if 500 ISE error is displayed |
| 12 | Click "Continue" | Repayment plan summary/confirmation page displayed. **FAIL** if 500 ISE error is displayed |
| 13 | Tick the "I agree for my payments to be processed... CPA" checkbox | Checkbox checked. **FAIL** if 500 ISE error is displayed |
| 14 | Click "Confirm Repayment Plan" | Redirected to card payment page. **FAIL** if 500 ISE error is displayed |
| 15 | Enter card details: number `4477 0000 0000 0006`, expiry `12/99`, CVV `111`, name `setup complete` | Card details entered and payment processed. **FAIL** if 500 ISE error is displayed |
| 16 | Refresh the page two times and click "Continue" | Page refreshed and continue completed. **FAIL** if 500 ISE error is displayed |
| 17 | Verify "You are currently on a Repayment Plan." text | Confirmation text visible. **FAIL** if 500 ISE error is displayed |
| 18 | Navigate to `/logout` | Logged out and redirected to homepage. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Borrower-initiated repayment plan setup works without errors
**Fail Criteria**: Page 404/500 ISE error on any step, or incorrect data displayed

---

## 3. Borrower can pay repayment plan installment

### TC-RP-005: Admin Login (for impersonation)
**Objective**: Admin login as a precondition to impersonate a borrower
**Priority**: Critical

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/login` | Login page displayed. **FAIL** if 500 ISE error is displayed |
| 2 | Enter the admin email from userdetails | Email field populated. **FAIL** if 500 ISE error is displayed |
| 3 | Enter the admin password from userdetails | Password field populated. **FAIL** if 500 ISE error is displayed |
| 4 | Click "Login" button | Redirect to `/admin/dashboard`. **FAIL** if 500 ISE error is displayed |
| 5 | Verify admin dashboard loaded | "The Money Platform" heading displayed. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Admin logged in
**Fail Criteria**: Login fails or 500 ISE error

---

### TC-RP-006: Borrower can pay repayment plan installment
**Objective**: Verify borrower can pay a repayment plan installment
**Priority**: High
**Preconditions**: Admin logged in (TC-RP-005)

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/admin/repayment-plans` | Repayment plans page with list of plans displayed. **FAIL** if 500 ISE error is displayed |
| 2 | Select a repayment plan and click "Details" | Plan details page displayed. **FAIL** if 500 ISE error is displayed |
| 3 | Verify application state | State badge displayed (e.g., "planned"). **FAIL** if 500 ISE error is displayed |
| 4 | Click "Accept Test" borrower name link, then "Toggle User Flag" | User flag toggled. **FAIL** if 500 ISE error is displayed |
| 5 | Click "Impersonate this user" | Redirected to borrower dashboard. **FAIL** if 500 ISE error is displayed |
| 6 | Verify "You are currently on a Repayment Plan." text | Text visible on borrower dashboard. **FAIL** if 500 ISE error is displayed |
| 7 | Verify "Make a Payment" button is visible and click it | Payment page displayed with amount + payment method options. **FAIL** if button missing or 500 ISE |
| 8 | Enter installment amount (e.g., £40) and complete the payment | Payment processed via card/bank. **FAIL** if 500 ISE error is displayed |
| 9 | Verify "Your payment was successful" text on dashboard | Success message visible. **FAIL** if 500 ISE error is displayed |
| 10 | Navigate to `/logout` | Logged out and redirected to homepage. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Borrower can pay repayment plan installment without errors
**Fail Criteria**: Page 404/500 ISE error on any step, or incorrect data displayed
