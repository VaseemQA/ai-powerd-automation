# Admin - Login Regression Test Cases

## Document Information
- **Product**: TMP Admin - Authentication
- **Test Environment**: UAT or INT (selected at run time)
- **Last Updated**: 2026-06-15
- **Test Type**: Regression Testing - Admin Login
- **User Role**: Admin / Super Admin

---

## Test Data Configuration

### Admin Credentials
See `config/userdetails.md` (Admin user).

### Invalid Credentials (for negative tests)
- **Invalid Email**: `notanadmin@example.com`
- **Invalid Password**: `WrongPass!1`
- **Malformed Email**: `not-an-email`

> All URLs in step tables are relative paths. The runner prefixes them with the
> selected environment's base URL (UAT or INT).

---

## 1. Admin can login with valid credentials

### TC-LGN-001: Admin login - happy path
**Objective**: Verify admin can successfully login to TMP admin panel with valid credentials
**Priority**: Critical
**Preconditions**: Valid admin credentials available in `config/userdetails.md`

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/login` | Login page displayed with Email, Password fields and Login button. **FAIL** if 500 ISE error is displayed |
| 2 | Verify page title and heading | Login page title and heading visible. **FAIL** if 500 ISE error is displayed |
| 3 | Enter the admin email from userdetails | Email field populated with admin email. **FAIL** if 500 ISE error is displayed |
| 4 | Enter the admin password from userdetails | Password field populated (masked). **FAIL** if 500 ISE error is displayed |
| 5 | Click "Login" button | Redirect to admin dashboard `/admin/dashboard`. **FAIL** if 500 ISE error is displayed |
| 6 | Verify admin dashboard loaded | "The Money Platform" heading displayed, Today's Payments section visible. **FAIL** if 500 ISE error is displayed |
| 7 | Verify admin user identity in header/menu | Logged-in admin user indicator visible. **FAIL** if 500 ISE error is displayed |

**Pass Criteria**: Admin logged in and dashboard accessible
**Fail Criteria**: Login fails, dashboard not accessible, or 500 ISE error on any step

---

## 2. Admin login - negative scenarios

### TC-LGN-002: Login fails with invalid email
**Objective**: Verify login is rejected when an unregistered email is used
**Priority**: High
**Preconditions**: User is not logged in

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/login` | Login page displayed. **FAIL** if 500 ISE error is displayed |
| 2 | Enter the invalid email `notanadmin@example.com` | Email field populated. **FAIL** if 500 ISE error is displayed |
| 3 | Enter the admin password from userdetails | Password field populated. **FAIL** if 500 ISE error is displayed |
| 4 | Click "Login" button | User remains on `/login` page; error message indicating invalid credentials is displayed. **FAIL** if redirect to dashboard occurs or 500 ISE error |
| 5 | Verify no admin session is established | URL still on `/login`; no admin dashboard content visible. **FAIL** if dashboard content rendered |

**Pass Criteria**: Login rejected with appropriate error; no session created
**Fail Criteria**: Dashboard accessible, no error shown, or 500 ISE error

---

### TC-LGN-003: Login fails with invalid password
**Objective**: Verify login is rejected when a wrong password is used for a valid admin email
**Priority**: High
**Preconditions**: User is not logged in

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/login` | Login page displayed. **FAIL** if 500 ISE error is displayed |
| 2 | Enter the admin email from userdetails | Email field populated. **FAIL** if 500 ISE error is displayed |
| 3 | Enter the invalid password `WrongPass!1` | Password field populated (masked). **FAIL** if 500 ISE error is displayed |
| 4 | Click "Login" button | User remains on `/login` page; error message indicating invalid credentials is displayed. **FAIL** if redirect to dashboard occurs or 500 ISE error |
| 5 | Verify no admin session is established | URL still on `/login`; no admin dashboard content visible. **FAIL** if dashboard content rendered |

**Pass Criteria**: Login rejected with appropriate error; no session created
**Fail Criteria**: Dashboard accessible, no error shown, or 500 ISE error

---

### TC-LGN-004: Login fails with empty email and password
**Objective**: Verify form validation prevents login when fields are empty
**Priority**: Medium
**Preconditions**: User is not logged in

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/login` | Login page displayed. **FAIL** if 500 ISE error is displayed |
| 2 | Leave Email field empty | Email field remains empty. **FAIL** if 500 ISE error is displayed |
| 3 | Leave Password field empty | Password field remains empty. **FAIL** if 500 ISE error is displayed |
| 4 | Click "Login" button | Validation error(s) displayed indicating required fields; user remains on `/login`. **FAIL** if redirect to dashboard occurs or 500 ISE error |

**Pass Criteria**: Required-field validation triggered; no session created
**Fail Criteria**: Dashboard accessible, no validation shown, or 500 ISE error

---

### TC-LGN-005: Login fails with malformed email
**Objective**: Verify email format validation rejects malformed addresses
**Priority**: Medium
**Preconditions**: User is not logged in

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Navigate to `/login` | Login page displayed. **FAIL** if 500 ISE error is displayed |
| 2 | Enter malformed email `not-an-email` | Email field populated. **FAIL** if 500 ISE error is displayed |
| 3 | Enter the admin password from userdetails | Password field populated. **FAIL** if 500 ISE error is displayed |
| 4 | Click "Login" button | Email format validation error displayed; user remains on `/login`. **FAIL** if redirect to dashboard occurs or 500 ISE error |

**Pass Criteria**: Email format validation triggered; no session created
**Fail Criteria**: Dashboard accessible, no validation shown, or 500 ISE error

---

## 3. Admin can logout

### TC-LGN-006: Admin logout
**Objective**: Verify admin can logout and the session is terminated
**Priority**: High
**Preconditions**: Admin logged in (from TC-LGN-001)

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | From admin dashboard, navigate to `/logout` | User logged out and redirected to homepage or `/login`. **FAIL** if 500 ISE error is displayed |
| 2 | Attempt to navigate to `/admin/dashboard` directly | User redirected back to `/login` (session terminated). **FAIL** if dashboard accessible without re-login or 500 ISE error |

**Pass Criteria**: Session terminated; protected pages require re-login
**Fail Criteria**: Dashboard accessible after logout, or 500 ISE error
