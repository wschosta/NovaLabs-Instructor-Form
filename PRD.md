# Program Requirements Document: NovaLabs Instructor FormBot

**Version:** v3.4.1
**Author:** Pecacheu
**License:** GNU GPL v3

---

## 1. Purpose

FormBot is a web application that automates the process of filling out instructor reimbursement forms at Nova Labs, a community makerspace. It streamlines the workflow of submitting class data, generating PDF summaries, and emailing completed forms to the appropriate accounting and membership contacts.

---

## 2. System Architecture

### 2.1 Overview

FormBot is a single-page web application with a Node.js backend and a vanilla JavaScript frontend. The server and client communicate in real time via Socket.IO. There is no database; state is held in memory on the server for the duration of active sessions.

### 2.2 Technology Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js (ES Modules) |
| Transport | HTTP/HTTPS server (Node built-in) |
| Real-time | Socket.IO v4 |
| Routing/Utilities | `raiutils` (router, schema, utils) |
| Email | Nodemailer (Gmail SMTP) |
| Authentication | OTP via `otplib` |
| PDF Generation | jsPDF (client-side) |
| HTML Sanitization | `string-strip-html` |
| Logging | `chalk` |

### 2.3 File Structure

```
server.js            — Node.js server: auth, API integration, email, Socket.IO
root/
  index.html         — Single-page HTML form interface
  form.js            — Client-side logic: UI, PDF gen, Socket.IO, background animation
  formStyle.css      — Styling with responsive breakpoints and custom fonts
  resources/
    bg.svg           — Tiling background image
    jspdf.min.js     — jsPDF library (vendored)
    type/            — Web font files (Open Sans, Roboto, Ubuntu Titling, Khmer UI)
config.json          — Secrets file (gitignored): API keys, mail password, OTP key, SSL paths
package.json         — Dependencies (no build step)
formbot.service      — Systemd unit file for production deployment
run.sh               — Launch script using GNU Screen
```

### 2.4 Client-Server Communication

All meaningful interaction between the frontend and backend occurs over **Socket.IO**, not REST endpoints. The only HTTP endpoint beyond static file serving is `POST /upload` for receipt file uploads. The Socket.IO protocol uses an event-based acknowledgment pattern where the server emits `ack` events back to the client with success/failure status.

---

## 3. Authentication & Session Management

### 3.1 OTP Authentication

- Clients must present a valid time-based one-time password (TOTP) token upon connection.
- The OTP secret is stored in `config.json` (`otpkey`).
- Tokens are verified server-side using `otplib`.
- Invalid tokens result in a delayed `badTkn` event (1-second delay to deter brute force).
- On successful verification, the server assigns a `crypto.randomUUID()` session ID.

### 3.2 Session Persistence

- The session ID is stored client-side in a cookie (`uid`) with a 4-hour lifetime.
- On reconnection, the client sends its stored ID instead of a new OTP token.
- The server validates the ID against its in-memory client map (`Cli`).
- Disconnected sessions are retained for 4 hours (`IDTimeout`) before cleanup.
- If the server version changes between connections, the client forces a page reload.

### 3.3 Wild Apricot API Authentication

- The server authenticates with Wild Apricot using an API key from `config.json`.
- OAuth2 client credentials grant is used against `https://oauth.wildapricot.org/auth/token`.
- The access token is cached in memory and automatically expires based on the token's `expires_in` value.
- Re-authentication happens transparently when the token expires.

---

## 4. Core Features

### 4.1 Event Auto-Fill (Portal Mode)

When the user selects "On Events Portal" and enters an event ID:

1. The client emits a `getEvent` Socket.IO event with the event ID.
2. The server fetches event details from the Wild Apricot API:
   - Event metadata (name, date, location, description, registration counts)
   - Registration list with attendee details
   - Individual contact records to determine membership level
   - Registration type to identify instructors (types starting with "5. instructor")
3. The server parses and returns a structured event object containing:
   - Event name, ID, portal link, venue, location
   - Date and time (formatted)
   - Fee information (highest base price across registration types)
   - Description (HTML stripped)
   - Host list (instructors) and RSVP list (attendees with name, email, membership level, fee paid)
4. The client auto-fills all form fields from the event data:
   - Class name, date, instructor name
   - Cost per student, student count
   - Attendee table rows with names, membership levels, and payment amounts
   - Class type is inferred from the event name (safety, sign-off, or project)

### 4.2 Ad-Hoc Mode

When the user selects "Ad-Hoc":
- All form fields become manually editable.
- No Wild Apricot API calls are made.
- The title is appended with `[ADHOC]` in the generated PDF.

### 4.3 Form Fields

| Field | Description | Validation |
|-------|------------|------------|
| Class Source | Portal event or Ad-Hoc | Required select |
| Event ID / Name | WA event ID (portal) or free text (ad-hoc) | Max 120 chars, regex pattern |
| Class Date | Date and time | Datetime-local input |
| Instructor Name | Submitting instructor | Text with char pattern |
| Email | Instructor's PayPal or contact email | Email pattern validated |
| Payment Method | PayPal, ADP, or Check | Select dropdown |
| Class Type | Project, Tool Sign-Off, or Safety Sign-Off | Select dropdown |
| Cost/Student | Per-student fee | Numeric, warns below $15 |
| Student Count | Number of attendees | Numeric 0-200 |
| Materials Cost | Direct class expenses | Numeric >= 0 |
| Receipt Files | Attached when materials > $0 | PDF, PNG, or JPEG; max 1GB total |
| NovaLabs Rate | Organization's revenue share percentage | Numeric 0-100, default 30% |

### 4.4 Attendee Table

- Dynamically generated rows based on student count.
- Each row contains: Name, Membership Level, Notes (dropdown), Payment amount.
- Notes dropdown options: No Show, Youth Under 18, Youth Robotics, Makerschool, Paid Using Donate Online, Do Not Sign Off.
- Auto-populated from RSVP data when using portal mode.
- Each attendee is sent to the server as a 4-element array: `[name, notesString, memberLevel, payment]`. The server splices elements 0 and 1 (`a.splice(0,2,a[0]+a[1])`) to produce a 3-element array: `[nameWithNotes, memberLevel, payment]`.

### 4.5 Cost Calculation

The PDF includes a cost breakdown:

```
Revenue - Materials = Profit
Profit - (NovaLabs Rate%) = Reimbursement (+ Materials if applicable)
Revenue - Reimbursement = Nova Labs Profit
```

### 4.6 PDF Generation

- Generated client-side using jsPDF.
- Portrait orientation, 8.5" x 11" format.
- **Page 1**: Class info header, all form fields, cost breakdown.
- **Page 2**: Attendee list table with name, membership level, and payment columns.
- Previewed in an iframe overlay before submission.
- Two-step submit: "Preview Summary" generates PDF, then "Submit PDF" sends it.
- Max PDF size: 20KB.

### 4.7 Receipt Upload

- Triggered when materials cost > $0.
- Uses a custom binary protocol over `POST /upload`:
  - First 4 bytes: header length (uint32 LE).
  - Next N bytes: JSON header array with `{n: filename, t: MIME type, l: byte length}` per file.
  - Remaining bytes: concatenated file data.
- Server validates the upload against the `raiutils/schema` checker.
- Upload is associated with the client's session by socket ID passed as a query parameter.
- Max upload size: 1GB.

### 4.8 Email Submission

When the form is submitted:

1. The server validates all input fields with regex patterns and type checks.
2. An HTML email is composed containing:
   - A header identifying it as an automated FormBot message.
   - An embedded event card (if event data is available) showing event details.
   - The attendee list rendered as a styled HTML table.
   - Material cost notation and sign-off type indicators.
   - A red warning banner if class type is Safety Sign-Off (`sType==2`): "No NovaPass or tool sign off. Safety Sign-Off Only."
3. The PDF is attached along with any receipt files.
4. The email is sent to:
   - The accounting relay address(es) (`AccAddr`).
   - The instructor's email address.
   - The membership relay address (`MemAddr`) if class type involves sign-offs.
5. A test mode exists: if the instructor email is `test@example.com`, the subject is prefixed with `<<FORMBOT_TEST>>`.

---

## 5. Frontend Details

### 5.1 Background Animation

- A full-screen `<canvas>` element renders a tiling SVG background.
- The background scrolls continuously at a configurable speed (`BgSpd`).
- A radial gradient overlay darkens the edges.
- A blur effect is applied behind the content box:
  - **Desktop**: Software box blur via `getImageData`/`putImageData` (CPU-based).
  - **Mobile**: CSS `backdrop-filter: blur(8px)` (GPU-accelerated).
- Animation pauses when the browser tab is blurred.

### 5.2 Responsive Design

Five breakpoints:
- Default: Content box at 80% width.
- `max-width: 900px`: Content box at 88% width.
- `max-width: 570px`: Full width, smaller fonts and table text.
- `max-width: 350px`: Narrows detail elements to 60% width.
- `max-width: 300px`: 70% zoom fallback.

### 5.3 Status Overlay

A full-screen overlay displays connection status messages ("Connecting...", "Connection To Server Lost!", "Bad Token") and blocks interaction until resolved.

### 5.4 Info Box

An animated notification bar within the form area shows errors and success messages with colored backgrounds and fade-in transitions.

### 5.5 Character Pattern Filtering

Input fields use a `charPattern` attribute with regex patterns to filter keystrokes in real time, preventing invalid characters from being typed.

### 5.6 URL Query Parameter Auto-Fill

Loading the page with a `?id=<eventId>` query parameter automatically switches to portal mode and triggers an event lookup for that ID on first connect. This is used by external tools (e.g., `wautils.nova-labs.org`) to deep-link directly to a specific event.

### 5.7 Built-in Test Mode

Calling `test()` in the browser console populates the form with mock data (fake event, fake attendees, test email) for development and testing purposes.

---

## 6. Server Details

### 6.1 Static File Serving

- The `raiutils/router` module serves static files from the `root/` directory.
- A virtual directory mapping (`VDir`) serves `utils.js` from the `raiutils` package.

### 6.2 Socket.IO Events

**Server-to-Client:**
| Event | Data | Purpose |
|-------|------|---------|
| `type` | — | Request client type identification |
| `badTkn` | — | Reject invalid authentication |
| `connection` | `(id, version)` | Confirm authenticated session |
| `ack` | `(eventType, success, data)` | Acknowledge client requests |

**Client-to-Server:**
| Event | Data | Purpose |
|-------|------|---------|
| `type` | `(clientType, token, id)` | Identify and authenticate |
| `getEvent` | `(eventId)` | Request event data from Wild Apricot |
| `sendForm` | `(title, date, name, email, materialCost, pdf, attendeeList, signOffType)` | Submit completed form |

### 6.3 Input Validation

All `sendForm` inputs are validated server-side:
- String fields: type check, length limit, regex pattern match.
- Numeric fields: type check, range validation.
- Attendee list: array type, max 200 entries, per-entry field validation.
- PDF: string type, max 20KB.
- Material cost > $0 requires receipt data to be uploaded first.
- Title must contain a colon and not be a plain number.

### 6.4 Server Console

An interactive stdin interface provides:
- `list`: Display all connected clients with their session info.
- `q` / `exit`: Graceful shutdown.

### 6.5 HTTPS Support

- Optional: enabled when `config.json` provides valid `key` and `cert` paths.
- Falls back to HTTP with a console warning if certificates cannot be loaded.

---

## 7. Configuration

### 7.1 config.json (Required, Gitignored)

```json
{
  "apikey": "Wild Apricot API key",
  "mailpass": "Gmail app password for SMTP",
  "otpkey": "TOTP shared secret",
  "key": "/path/to/ssl/private.key",
  "cert": "/path/to/ssl/certificate.crt"
}
```

### 7.2 Hardcoded Constants (server.js)

| Constant | Value | Description |
|----------|-------|-------------|
| `Port` | 8080 | HTTP(S) listening port |
| `SendTimeout` | 15000ms | Email send timeout |
| `ReqTimeout` | 5000ms | HTTPS request timeout |
| `IDTimeout` | 4 hours | Disconnected session retention |
| `MaxUpload` | 1GB | Maximum receipt upload size |
| `MailHost` | formbot@nova-labs.org | SMTP sender address |
| `AccAddr` | `['formbot-events-relay@nova-labs.org']` | Accounting recipient(s) (array) |
| `MemAddr` | formbot-membership-relay@nova-labs.org | Membership recipient |

---

## 8. Deployment

### 8.1 Prerequisites

- Node.js and NPM installed.
- `config.json` with valid credentials.
- Network access to Wild Apricot API and Gmail SMTP.

### 8.2 Installation

```bash
npm install
node server
```

### 8.3 Production

- Systemd service file (`formbot.service`) for automatic startup and restart. Note: the `ExecStart` path is hardcoded to the developer's home directory and must be updated for other deployments.
- Runs inside a GNU Screen session via `run.sh` for console access.
- Service type is `forking` with `Restart=always`.

---

## 9. Dependencies

| Package | Purpose |
|---------|---------|
| `chalk` ^5.6.2 | Colored console logging |
| `nodemailer` ^8.0.1 | SMTP email delivery |
| `otplib` ^13.2.1 | TOTP authentication |
| `raiutils` ^8.7.11 | HTTP router, schema validation, client-side utilities |
| `socket.io` ^4.8.3 | Real-time WebSocket communication |
| `string-strip-html` ^13.5.3 | Strip HTML from event descriptions |
| `jspdf` (vendored) | Client-side PDF generation |

---

## 10. External Service Integration

### 10.1 Wild Apricot

- **Auth endpoint**: `https://oauth.wildapricot.org/auth/token`
- **API base**: `https://api.wildapricot.org/v2/accounts/{accountId}`
- **Used endpoints**:
  - `GET /events/{eventId}` — Event details
  - `GET /eventregistrations?eventId={eventId}` — Registration list
  - `GET /contacts/{contactId}` — Member contact and level info

### 10.2 Gmail SMTP

- **Host**: smtp.gmail.com
- **Port**: 587 (STARTTLS)
- **Auth**: App password from config
