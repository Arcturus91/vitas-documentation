# Phase 1 Implementation - Functional Validation Checklist

**Purpose:** Validate Phase 1 implementation by testing user-facing functionality
**Focus:** What can users ACCOMPLISH (not technical implementation)
**How to use:** Check each box when the functionality is working as described

---

## 1️⃣ AUTHENTICATION & ACCOUNT REGISTRATION

### Owner Account Creation
- [ ] Register a new account with email and password
- [ ] Receive verification email within 1 minute
- [ ] Click verification link and see confirmation message
- [ ] Optionally enable MFA (OTP via email or WhatsApp)
- [ ] Accept Terms & Conditions with electronic signature
- [ ] See confirmation that my clinic/IPRESS account was created
- [ ] Account automatically transitions from Draft → Incomplete → Active states
- [ ] View audit log showing my registration with IP and timestamp

### Doctor Authentication
- [ ] Log in with email and password (invited doctor)
- [ ] Stay logged in for 24 hours without re-authentication
- [ ] See HTTP-only cookie set in browser (check DevTools)
- [ ] Get logged out automatically after 24 hours

### Patient Authentication
- [ ] Enter phone number and request OTP
- [ ] Receive 6-digit OTP via WhatsApp within 30 seconds
- [ ] Enter OTP and get authenticated
- [ ] See error if OTP expires after 5 minutes
- [ ] Stay authenticated for 24 hours during payment flow

### Chatbot Authentication
- [ ] Send WhatsApp message to clinic bot
- [ ] Receive response without manual login
- [ ] Bot maintains session for up to 1 hour
- [ ] Bot automatically refreshes authentication if JWT expires
- [ ] Session expires after 15 minutes of inactivity

---

## 2️⃣ ACCOUNT SETTINGS MODULE

### Account Profile
- [ ] View my organization's profile information
- [ ] Edit my city/district (but cannot edit country or state - locked)
- [ ] See timezone auto-set by country (read-only)
- [ ] Change language preference (default: Spanish)
- [ ] See date format auto-set internally (not user-changeable)
- [ ] Click "Clear data" button and see warning modal
- [ ] Close account and see confirmation that clinical records are preserved

### Account Security
- [ ] Change my password
- [ ] Update contact information
- [ ] View audit history of all account changes
- [ ] See each audit entry with: date, action, user, IP address

---

## 3️⃣ USERS & PERMISSIONS MODULE

### User List
- [ ] View list of all users I've created
- [ ] See for each user: Name, Email, Role, Status, Last Login
- [ ] See user status indicators: 🟠 Invited, 🟡 Incomplete, 🟣 Pending, 🟢 Active, 🔴 Inactive

### Doctor Invitation Flow (As Owner)
- [ ] Click "Invite User" button
- [ ] Enter doctor's full name (required)
- [ ] Enter doctor's email (required, validated format)
- [ ] Select role (default: Clinical)
- [ ] See confirmation modal explaining the invitation process
- [ ] See warning: "Link expires in 24 hours, do not share"
- [ ] Click confirm and see "Invitation sent" success message
- [ ] Verify doctor appears in user list with 🟠 Invited status

### Invitation Email (As Doctor - Received)
- [ ] Receive personalized email: "Hello, Dr. [Name]"
- [ ] See clear explanation of invitation from [Clinic Name]
- [ ] Click "Activate Account" button
- [ ] See 24-hour expiration notice
- [ ] See security warning about not sharing the link

### Activation Screen (As Doctor)
- [ ] Land on activation page from email link
- [ ] Create password (min 8 chars, 1 uppercase, 1 lowercase, 1 number)
- [ ] Confirm password with validation
- [ ] See password strength indicator (Weak/Medium/Strong)
- [ ] Check "Accept Terms & Privacy Policy" checkbox
- [ ] Click link to read Terms & Privacy Policy
- [ ] Click "Activate Account" button
- [ ] Auto-login without manual credentials entry
- [ ] Get redirected to Clinical Profile page to complete professional data
- [ ] See status change from 🟠 Invited → 🟡 Incomplete

### Cancel Activation
- [ ] Click "Cancel Activation" button
- [ ] See warning modal about consequences
- [ ] Confirm cancellation and return to previous page

---

## 4️⃣ CLINICAL PROFILE OF DOCTOR

### Professional Profile Completion (As Doctor in Onboarding)
- [ ] See toast message: "Account activated. Complete your professional profile."
- [ ] Sidebar and owner modules are hidden (focused onboarding mode)
- [ ] Profile opens in edit mode by default

#### Personal Data Section
- [ ] See full name (read-only, from invitation)
- [ ] Select document type (DNI/CE/Passport)
- [ ] Enter document number
- [ ] Enter date of birth and see age auto-calculated
- [ ] Select gender
- [ ] Enter address
- [ ] Enter personal phone number
- [ ] Enter emergency contact phone
- [ ] See email (read-only, from invitation)
- [ ] Upload digital signature (PNG image)

#### Professional Data Section
- [ ] Select profession (Doctor, Nurse, Obstetrician, etc.)
- [ ] Enter CMP (required if profession = Doctor)
- [ ] Enter RNE (if applicable)
- [ ] Select primary specialty from dropdown
- [ ] Optionally select subspecialty
- [ ] Select primary work location (required)
- [ ] Optionally select secondary work location
- [ ] See validation error if CMP missing for Doctor profession

### Save and State Transition
- [ ] Click "Save" button
- [ ] See success message
- [ ] See status badge change from 🟡 Incomplete → 🟣 Pending
- [ ] Cannot schedule appointments yet (awaiting admin validation)

### Owner Validation (As Owner)
- [ ] View doctor profile with 🟣 Pending status
- [ ] Review all professional information
- [ ] Click "Verify Profile" button
- [ ] See confirmation modal
- [ ] Confirm validation
- [ ] See status change from 🟣 Pending → 🟢 Active
- [ ] Verify doctor can now be scheduled for appointments
- [ ] See audit log entry with my IP and timestamp

### Normal Mode (As Active Doctor)
- [ ] Log in and see full platform layout (sidebar visible)
- [ ] Navigate to Clinical Profile
- [ ] Profile opens in read-only mode
- [ ] Click "Edit" button to modify information
- [ ] Make changes and click "Save"
- [ ] Profile returns to read-only mode

---

## 5️⃣ LEGAL DOCUMENTS & CONSENTS

### Document Management (As Owner)
- [ ] Create new institutional legal document
- [ ] Enter custom title (e.g., "Privacy Policy")
- [ ] Enter public URL or upload signed PDF
- [ ] Enter version number and effective date
- [ ] Save document
- [ ] Download VITAS base template
- [ ] See disclaimer about legal review requirement

### Document Versioning
- [ ] Update existing document version
- [ ] See all previous consents marked as "Outdated"
- [ ] Verify system tracks: policyUri, version, hash for each document

### Patient Consent Flow - Manual Mode (As Doctor)
- [ ] Download consent document
- [ ] Print for patient to sign physically
- [ ] Patient signs physical document
- [ ] Upload signed document to system
- [ ] See consent status: ✅ Verified (manual)

### Patient Consent Flow - Intelligent Mode (As Patient via WhatsApp)
- [ ] Receive consent document links from AI assistant via WhatsApp
- [ ] Click links to review documents
- [ ] Accept digitally via simple electronic signature
- [ ] See consent status: ✅ Accepted (intelligent)

### Consent States
- [ ] See consent states clearly indicated:
  - ❌ Not Accepted
  - ✅ Accepted
  - ✗ Revoked
  - ● Outdated
- [ ] Verify that when consent is "Not Accepted", patient cannot receive external messages
- [ ] Verify that when consent is "Revoked", all external messaging stops immediately
- [ ] Verify that when consent is "Outdated", AI requests re-acceptance

### Audit & Traceability
- [ ] View audit log for all consent acceptances
- [ ] See for each: userId, terms_version, hash_terms, IP, timestamp, action
- [ ] Verify WORM storage (files cannot be modified once saved)

---

## 6️⃣ OPERATIONAL PARAMETERS

### Appointment Policies Configuration (As Owner)
- [ ] Set maximum late arrival tolerance (minutes)
- [ ] Set free rescheduling allowance (count)
- [ ] Set penalty for late cancellation (%)
- [ ] Set no-show limit before blocking (count)
- [ ] Set refund administrative fee (%)
- [ ] Set refund request deadline (hours post-appointment)
- [ ] Set credit validity after no-show (days)
- [ ] Set extra recording time tolerance (minutes after appointment)
- [ ] Save all parameters

### Communication Channels
- [ ] Enable/disable WhatsApp (checkbox)
- [ ] Enable/disable SMS (checkbox)
- [ ] Enable/disable Email (checkbox)
- [ ] Save channel preferences

### Post-Appointment Reminders
- [ ] Set hours after appointment to send follow-up
- [ ] Configure sender name/signature visible in messages
- [ ] Edit message template with dynamic variables ({name}, {hour}, {doctor})
- [ ] Preview message with sample data

### VITAS Default Policies
- [ ] Click "Use Default VITAS Policies" button
- [ ] See fields populate with VITAS defaults (locked)
- [ ] See version notice: "Using VITAS base configuration v1.x"
- [ ] See disclaimer about accepting system-defined conditions

### Change History
- [ ] View change history table
- [ ] See for each change: Date, User, Field Changed, Old Value, New Value
- [ ] Verify history is stored in WORM format (immutable)

---

## 7️⃣ MARKETING & COMMUNICATION

### Campaigns
- [ ] Create new campaign
- [ ] Enter campaign title
- [ ] Set validity dates (start and end)
- [ ] Select linked services (multi-select from clinic services)
- [ ] Enter campaign information (terms, prices, restrictions)
- [ ] See expired campaigns displayed in red
- [ ] Verify AI assistant uses campaign info when responding to patients

### Social Media Links
- [ ] Add Facebook link
- [ ] Add Instagram link
- [ ] Add TikTok link
- [ ] Add LinkedIn link
- [ ] Add custom social network ("Other" option with custom name)
- [ ] Verify AI assistant references these links when suggesting patients visit social media

### AI Virtual Assistant Configuration
- [ ] Upload assistant avatar photo
- [ ] Enter personalized assistant name
- [ ] Enter WhatsApp business phone number
- [ ] Generate webchat link
- [ ] Copy webchat link
- [ ] Generate QR code for webchat
- [ ] Download QR in PNG format
- [ ] Download QR in SVG format
- [ ] Download QR in PDF format
- [ ] Test webchat link in browser
- [ ] See "Powered by Vitas" footer in webchat
- [ ] See consent banner in webchat
- [ ] Complete quick OTP identification in webchat
- [ ] Register and book appointment via webchat
- [ ] Make payment via webchat

---

## 8️⃣ HUMAN SUPPORT CONFIGURATION

### Support Contact Setup (As Owner)
- [ ] Enter contact name/role (e.g., "Patient Support Area")
- [ ] Set human support hours (e.g., Mon-Fri 8:00-18:00)
- [ ] Enter support email address
- [ ] Enter support phone/WhatsApp number
- [ ] Enter support web URL/form link
- [ ] Save configuration

### AI Assistant Handoff Behavior (As Patient)
- [ ] Type "human" in WhatsApp chat
- [ ] See AI offer human support options
- [ ] See only configured channels (empty fields not offered)
- [ ] Request human support outside hours
- [ ] See message: "Our human team will respond during: Mon-Fri 8:00-18:00"

### Audit Logging
- [ ] View audit log of all handoff requests
- [ ] See for each: date/time, request type, channel, user, reason

---

## 9️⃣ MESSAGING & REMINDERS

### Configuration (As Owner)
- [ ] Set number of reminders before appointment (e.g., 2)
- [ ] Set send intervals (e.g., 24h, 1h before appointment)
- [ ] Set allowed send hours (e.g., 08:00-21:00)
- [ ] Select authorized channels (WhatsApp, SMS, Email)
- [ ] Set post-appointment reminder time (hours after)
- [ ] Configure sender name identity
- [ ] Edit base message template with variables
- [ ] Save messaging configuration

### AI Assistant Messaging Behavior (As Patient)
- [ ] Receive messages in my initial language (automatic detection)
- [ ] Receive pre-appointment reminders at configured intervals
- [ ] NOT receive messages if consent is "Not Accepted"
- [ ] NOT receive messages outside configured send hours
- [ ] NOT receive messages if consent is "Revoked"
- [ ] Verify pending messages are canceled if I cancel/reschedule appointment

### Message Types Received
- [ ] Pre-appointment reminder (24h before)
- [ ] Pre-appointment reminder (1h before)
- [ ] Post-appointment follow-up (results/satisfaction survey)
- [ ] Administrative alert (pending payment)
- [ ] Administrative alert (no-show penalty)
- [ ] Support communication (human handoff confirmation)

### Message History (As Owner/Admin)
- [ ] View message log table
- [ ] See for each message: Date, Type, Channel, Status, Actor, Result
- [ ] Filter by patient
- [ ] Filter by date range
- [ ] Filter by channel
- [ ] Filter by message type
- [ ] See read-only table (cannot edit history)
- [ ] Verify 12-month minimum retention
- [ ] Verify WORM storage (immutable records)

---

## 🔟 SERVICES - MEDICAL SPECIALTIES

### Specialty Creation (As Owner - Pre-Creation Mode)
- [ ] Navigate to Services module
- [ ] Click "+ Add Specialty"
- [ ] Start typing specialty name in autocomplete field
- [ ] See suggestions while typing
- [ ] See option "Create '[text]'" if no matches
- [ ] Select or create specialty
- [ ] See "Specialty saved ✓" confirmation
- [ ] See specialty with 🔵 Planned (blue) status

### Anti-Duplication
- [ ] Try creating duplicate specialty
- [ ] See suggestion: "Use existing 'Dermatology'?"
- [ ] Verify normalization (singular, standard capitalization)

### Specialty States Visualization
- [ ] See 🔵 Planned specialty (created, no doctors, not schedulable)
- [ ] Assign first doctor to specialty
- [ ] See specialty auto-transition to 🟢 Active (green, schedulable)
- [ ] Deactivate specialty
- [ ] See specialty change to 🔴 Inactive (red, paused, no new appointments)

### Configure Specialty Parameters
- [ ] Click specialty card
- [ ] Open detail view
- [ ] Click "Edit Parameters" button
- [ ] See specialty name (read-only)
- [ ] Edit appointment duration (10, 15, 20, 30 min)
- [ ] Edit buffer between appointments (0-30 min)
- [ ] Edit base consultation price
- [ ] See automatic fields (read-only):
  - Late = 30% of duration (auto-calculated)
  - No-show = 100% of duration (auto-calculated)
- [ ] See preview of Late/No-show times when changing duration
- [ ] See validation: duration 5-min steps, minimum 5 min
- [ ] See validation: buffer 0-30 min, integers
- [ ] See hint: "Buffer does not generate schedulable slots"
- [ ] Click "Save Changes"
- [ ] See warning if doctors already assigned: "⚠ This change will impact all doctors"
- [ ] Confirm and save
- [ ] Verify parameters apply immediately to all doctors in specialty

### Deactivate Specialty
- [ ] Open specialty detail
- [ ] Click "Deactivate Specialty" button (red outline)
- [ ] See confirmation modal with consequences:
  - "No new appointments can be created"
  - "Doctors will stop receiving bookings"
  - "History will be preserved"
- [ ] Confirm deactivation
- [ ] See status change to 🔴 Inactive
- [ ] Verify no new appointments can be created for this specialty
- [ ] Verify historical data is intact
- [ ] See audit log entry with user, date/time, reason

### Reactivate Specialty
- [ ] Filter list by Inactive specialties
- [ ] Select inactive specialty
- [ ] Open detail view
- [ ] Click "Reactivate" button
- [ ] See confirmation modal
- [ ] Confirm reactivation
- [ ] See status return to 🟢 Active
- [ ] Verify all previous parameters restored (duration, buffer, price)
- [ ] Verify doctors regain specialty in profiles
- [ ] See audit log entry

### Delete Specialty (Unused Only)
- [ ] Create specialty with no doctors or appointments
- [ ] See "Delete Permanently" option available
- [ ] See double confirmation modal
- [ ] Confirm deletion
- [ ] Verify specialty removed from system
- [ ] Try to delete specialty with history
- [ ] See error: "Cannot delete, only deactivate"

### Specialty Visibility
- [ ] View Health Professionals module
- [ ] See only Active specialties listed
- [ ] Verify Planned and Inactive specialties not shown

### AI Assistant Behavior (As Patient)
- [ ] Try to book appointment in Inactive specialty via WhatsApp
- [ ] See message: "This specialty is no longer available. Would you like other options?"

---

## 1️⃣1️⃣ SERVICES - PROCEDURES

### Add Procedure to Global Catalog (As Owner)
- [ ] Navigate to Services → Procedures
- [ ] Click "+ Add Procedure"
- [ ] Search for procedure in autocomplete field (filters by name)
- [ ] Select procedure from VITAS master catalog
- [ ] Enter standard duration (minutes)
- [ ] Enter base price (S/)
- [ ] See modality auto-set to "In-Person" (Phase 1 only)
- [ ] Select location/site where procedure is offered
- [ ] Optionally enter IPRESS Internal Code (unique, alphanumeric)
- [ ] Click "Save"
- [ ] See procedure added to clinic's global catalog

### Procedure Availability Rules
- [ ] Verify procedure is available for ALL doctors (no specialty restriction)
- [ ] Open appointment detail as doctor
- [ ] Search for procedure
- [ ] See procedures relevant to my specialty shown first
- [ ] Search entire global catalog if procedure not in suggested list

### AI Scribe Suggestions
- [ ] Complete appointment with AI Scribe enabled
- [ ] See AI suggest relevant procedures
- [ ] Accept AI suggestion
- [ ] Edit suggested procedure
- [ ] Delete suggested procedure

### Internal Code Validation
- [ ] Enter IPRESS Internal Code
- [ ] Try duplicate code
- [ ] See error: "Code must be unique"
- [ ] Save procedure with valid code
- [ ] Try editing code after billing exists
- [ ] See warning: "Requires controlled change flow"

### Edit Procedure
- [ ] Select procedure from list
- [ ] Open detail modal
- [ ] See read-only fields:
  - Procedure name
  - VITAS.code
  - ichi_code, ichi_display
  - ieds_code, ieds_display
  - Usage history count
- [ ] Edit standard duration
- [ ] Edit base price
- [ ] Edit location
- [ ] See warning if procedure already used: "⚠ Changes will impact all doctors and future appointments"
- [ ] Click "Save Changes"

### Deactivate Procedure
- [ ] Open procedure detail
- [ ] Click "Deactivate" button
- [ ] See confirmation modal with consequences:
  - "Cannot be selected in new appointments"
  - "Internal billing disabled"
  - "History preserved"
- [ ] Confirm deactivation
- [ ] See status change to 🔴 Inactive
- [ ] Try selecting in new appointment
- [ ] See notice: "This procedure is inactive. Can be indicated for external performance."
- [ ] Select anyway and see marked as "Requested (outside IPRESS)"
- [ ] Verify no internal execution/billing enabled
- [ ] See audit log entry

### Reactivate Procedure
- [ ] Filter by Inactive procedures
- [ ] Select inactive procedure
- [ ] Click "Reactivate" button
- [ ] See confirmation modal
- [ ] Confirm reactivation
- [ ] See status return to 🟢 Active
- [ ] Verify previous parameters restored (duration, price, location, code)
- [ ] Verify available again in appointment detail
- [ ] See audit log entry

### Delete Procedure (Unused Only)
- [ ] Create procedure never used in appointments/orders/billing
- [ ] See "Delete Permanently" option
- [ ] See double confirmation
- [ ] Confirm deletion
- [ ] Verify removed from system
- [ ] Try deleting procedure with history
- [ ] See error: "Cannot delete, only deactivate"

### Procedures Outside IPRESS (Clinical Freedom)
- [ ] As doctor, search for any procedure from VITAS Master Catalog
- [ ] Select procedure not active in clinic's IPRESS
- [ ] See marked in clinical history as "Requested (outside IPRESS)"
- [ ] See message to patient: "Not available at this clinic. Can be performed elsewhere."
- [ ] Verify traceability maintained even though not offered in clinic

### Empty States & Messages
- [ ] Enter Procedures with no active specialties
- [ ] See notice: "To operate procedures, first activate at least one specialty" with [Go to Specialties] button
- [ ] Search for non-existent procedure
- [ ] See: "No procedures found for '[text]'. Try another term."
- [ ] Simulate catalog load failure
- [ ] See: "Could not load catalog. Retry later."

---

## 1️⃣2️⃣ HEALTH PROFESSIONALS MODULE

### Professional List (As Owner)
- [ ] View "Health Professionals" page
- [ ] See H1: "Health Professionals"
- [ ] See search bar: "Search professionals"
- [ ] See "+ Add Professional" button (centered)
- [ ] See H2: "Professional List"
- [ ] See specialties as section dividers (headings)
- [ ] See professional cards within each specialty heading
- [ ] Verify headings remain fixed on scroll

### Professional Card Content
- [ ] See name & surname (bold)
- [ ] See CMP formatted as "CMP 12345" (if doctor)
- [ ] See location: "Location: San Vitas Surco"
- [ ] See photo (or placeholder if none)
- [ ] No "Active" chip visible (list shows only active by default)
- [ ] Tap card and navigate to Professional Profile

### Search Functionality
- [ ] Type doctor's first name (e.g., "Ana")
- [ ] See results after 2 characters
- [ ] See debounce delay (250ms)
- [ ] Type doctor's surname (e.g., "García")
- [ ] See matching results
- [ ] Type CMP number (e.g., "12345")
- [ ] See exact CMP matches shown first
- [ ] Type partial CMP (e.g., "123")
- [ ] See results with CMP containing "123"
- [ ] Try numeric search with "CMP 1234" format
- [ ] Try numeric search with "1234" (no prefix)
- [ ] See both return results
- [ ] Type multi-word query (e.g., "ana garcia")
- [ ] See results matching both tokens (AND operator)
- [ ] Type non-existent name
- [ ] See message: "No professionals found with '[text]'"
- [ ] Clear search with ✕ button
- [ ] See default list restored

### Search Performance
- [ ] Search with less than 2 characters
- [ ] Verify search not activated
- [ ] Load list with 50+ professionals
- [ ] Verify results limited to 50 items (virtualization)
- [ ] Verify search is case-insensitive
- [ ] Verify search is accent-insensitive
- [ ] Verify results grouped by specialty (headings maintained)
- [ ] Verify results ordered alphabetically by surname within each specialty

### Add Professional (Phase 1)
- [ ] Click "+ Add Professional" button
- [ ] Get redirected to Account Settings → Users & Permissions → + Add User
- [ ] See Role pre-set to "Clinical"
- [ ] Complete invitation flow
- [ ] Verify professional appears in list after activation

### Professional Profile - Navigation
- [ ] Click professional card from list
- [ ] Navigate to Professional Profile page
- [ ] See two main sections: Personal Data, Professional Data

### Professional Profile - Personal Data Section
- [ ] See photo and full name at top
- [ ] See masked document (e.g., "--123")
- [ ] See document type/number
- [ ] See date of birth with calculated age
- [ ] See gender
- [ ] See address
- [ ] See personal phone (read-only)
- [ ] See personal email (read-only)
- [ ] See emergency contact phone (editable by admin)
- [ ] Click pencil icon at bottom of Personal Data section
- [ ] Section enters edit mode (Professional Data stays read-only)
- [ ] Edit emergency contact phone
- [ ] Click "Save" button within Personal Data section
- [ ] See success message
- [ ] Section returns to read-only
- [ ] Try navigating away with unsaved changes
- [ ] See confirmation to save/discard

### Professional Profile - Professional Data Section
- [ ] See CMP (if doctor)
- [ ] See RNE (if exists)
- [ ] See "View professional's schedule" link
- [ ] Click and navigate to schedule view
- [ ] See "Edit user access" link
- [ ] Click and navigate to Users & Permissions
- [ ] See professional summary:
  - Profession
  - Active specialties (comma-separated)
  - Primary work location
  - Secondary work locations
  - Digital signature graphic (PNG image)
- [ ] If all roles inactive, see banner: "No active roles. Cannot receive new appointments."

### Professional Roles (PractitionerRole) Management
- [ ] See role list table/cards showing:
  - Specialty
  - Location
  - Status (Active/Inactive/Suspended by specialty)
  - Schedule (Configure button)
  - Action (Deactivate button)
- [ ] Click "Configure Schedule" for a role
- [ ] See block/pause configuration (duration from Specialty)
- [ ] Click "+ Add Role" button
- [ ] Select specialty from catalog (Active only)
- [ ] Location auto-assigned (mono-location Phase 1)
- [ ] Click "Save"
- [ ] See new PractitionerRole created with active=true
- [ ] See audit log entry: professional.role.add

### Deactivate Role
- [ ] Click "Deactivate" button in role row
- [ ] See confirmation: "Deactivate this role (Specialty/Location). Professional will stop receiving new appointments."
- [ ] Confirm deactivation
- [ ] See PractitionerRole.active=false
- [ ] Verify role no longer appears in Health Professionals list (Phase 1 shows only active)
- [ ] Verify historical data intact

### Role States & Dependencies
- [ ] Deactivate specialty globally
- [ ] See all roles for that specialty become "Suspended by specialty"
- [ ] See "Deactivate" button disabled for suspended roles
- [ ] Verify account blocking (Settings → Security) does NOT change PractitionerRole.active

### Cross Links
- [ ] Click "View account security"
- [ ] Navigate to Settings → Security
- [ ] Return to Professional Profile
- [ ] Click specialty name
- [ ] Open specialty detail (duration/buffers/base price)

### Empty States & Errors
- [ ] View professional with no roles
- [ ] See CTA "+ Add Role (specialty/location)"
- [ ] Try adding role with Inactive specialty
- [ ] See error message preventing addition
- [ ] Simulate save error
- [ ] See error banner with "Retry" option
- [ ] Verify local changes preserved after error

---

## ✅ IMPLEMENTATION VALIDATION COMPLETE

**When all checkboxes above are checked:**
- Phase 1 MVP is functionally complete from user perspective
- All 12 modules are operational
- Users can accomplish all specified tasks
- System is ready for Phase 2 planning

**Note:** This checklist focuses on FUNCTIONALITY (what users can do), not technical implementation (APIs, databases, Lambda functions). Use this to validate the product works as designed, regardless of how it's built under the hood.

---

**Last Updated:** 2025-11-08
**Total Acceptance Criteria:** 400+ user-facing validation points
