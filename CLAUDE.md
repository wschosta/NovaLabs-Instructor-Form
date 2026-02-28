# CLAUDE.md — Development Guidelines for NovaLabs Instructor FormBot

## Project Overview

FormBot is a Node.js web application that automates instructor reimbursement forms for Nova Labs. It uses Socket.IO for real-time client-server communication, integrates with the Wild Apricot API for event data, generates PDFs client-side with jsPDF, and sends completed forms via email through Gmail SMTP.

**Read `PRD.md` before making any changes.** It is the authoritative reference for how this application works and what it is intended to do.

---

## Architecture — Do Not Alter

The fundamental architecture of this codebase must be preserved. Specifically:

- **`server.js`** is the single backend entry point. Do not split it into multiple server files or introduce a framework (Express, Fastify, etc.) unless explicitly directed.
- **`root/form.js`** is the single client-side script. Do not introduce a build step, bundler, or frontend framework (React, Vue, etc.).
- **`root/index.html`** is a single-page HTML file with inline structure. Do not convert it to a templating system.
- **Socket.IO** is the transport layer for all form interactions. Do not replace it with REST endpoints or another WebSocket library.
- **The `POST /upload` endpoint** uses a custom binary protocol for receipt files. Do not change this to multipart/form-data or another format without explicit direction.
- **Client-side PDF generation** via jsPDF is intentional. Do not move PDF generation to the server.
- **No database.** Session state is held in memory. Do not introduce a database unless explicitly required.
- **ES Modules** (`"type": "module"` in package.json). Do not convert to CommonJS.

When in doubt, consult `PRD.md` for the intended behavior before making changes.

---

## Development Workflow

### Progressive Commits

Make small, incremental commits as you work. Do not batch all changes into a single large commit at the end.

1. **Commit after each logical unit of work** — a single feature, a single bug fix, or a single refactor.
2. **Write clear commit messages** that describe *what* changed and *why*.
3. **Do not mix concerns** — a commit that adds a feature should not also refactor unrelated code.

### Testing Before Commits

Before committing any change, verify it works:

1. **Server starts cleanly**: Run `node server` (requires `config.json` — if unavailable, at least verify there are no syntax errors with `node --check server.js`).
2. **Client-side changes**: Open the form in a browser and verify:
   - The page loads without console errors.
   - The modified feature works as expected.
   - Existing features are not broken (form fields, event auto-fill, PDF generation, submit flow).
3. **Use the built-in test mode**: Type `test()` in the browser console to populate the form with test data and verify the full flow.
4. **Check validation**: If you changed input handling, verify both valid and invalid inputs are handled correctly on client and server.

### Before Pushing

- Verify `node --check server.js` passes (no syntax errors).
- Verify `node --check root/form.js` passes (no syntax errors in strict mode).
- Review your diff to make sure no secrets, debug code, or unintended changes are included.

---

## Code Style

This codebase uses a compact, terse coding style. Follow the existing conventions:

- **Minimal whitespace**: Short variable names, compact expressions, semicolons on the same line.
- **No trailing semicolons on `function` declarations** — but use them in expression statements.
- **Single-letter variables** are common for loop counters, temporary values, and short-lived references (e.g., `d`, `e`, `r`, `f`, `s`).
- **Abbreviated names** for internal identifiers (e.g., `sck` for socket, `ev` for event, `hdr` for header, `adr` for address).
- **Inline conditionals** and short-circuit logic are preferred over verbose if/else blocks.
- **CSS class names** use camelCase (e.g., `muEvent`, `muTitle`, `muDesc`).
- **No JSDoc or type annotations.** Comments are used sparingly and only where logic is non-obvious.

Do not reformat or restyle existing code. Match the surrounding style when adding new code.

---

## Key Technical Details

### Server (`server.js`)

- Entry point: `await begin()` at the bottom of the file.
- Authentication flow: OTP verification → session UUID assignment → Socket.IO event handlers.
- Wild Apricot API: OAuth2 client credentials, token cached in `ATkn` with auto-expiry.
- `Debug` flag (default `0`): when non-zero, enables router debug logging, logs client list on new connections, and attaches raw API responses to event data as `evm.raw`.
- Email: Nodemailer transport via Gmail SMTP on port 587.
- File upload: Custom binary format — 4-byte header length (uint32 LE), JSON header array, concatenated file data.
- All `sendForm` inputs are validated server-side with regex patterns and type checks before email is composed.
- The `ack` function is the standard response mechanism: `ack(socket, eventType, dataOrError)`. If the third argument is a string, it's an error; otherwise (object, `undefined`, etc.) it's a success. The client receives `('ack', eventType, booleanSuccess, dataOrError)`.

### Client (`root/form.js`)

- `formLoad()` is the entry point (called on `window.onload`).
- Background animation runs via `requestAnimationFrame` loop.
- Two-step submit: first click generates PDF preview, second click uploads receipts + submits form data.
- `EvData` holds the currently loaded event (null in ad-hoc mode).
- `PdfData` holds the generated PDF buffer (falsy until preview is generated).
- `SData` holds session state (`id`, `v` version, `c` connected flag). The `c` flag gates first-connect logic (e.g., URL auto-fill) so it only runs once.
- `QData` holds parsed URL query parameters from `location.search`. If `QData.id` is present, the form auto-fills from that event ID on first connect.
- `rstForm()` resets the submit state when any field is modified after preview.

### Validation Patterns

```
pTitle = /^[\w\-:.<>()[\]&*%!', ]+$/   — Class titles
pText  = /^[\w\+\-()'. ]+$/             — Names and text fields
pEmail = /^\w+(?:[\.+-]\w+)*@\w+(?:[\.-]\w+)*\.\w\w+$/  — Email addresses
pDate  = /^[\w,: ]+$/                   — Formatted date strings
```

These patterns are enforced both client-side (via `charPat` keystroke filtering) and server-side (in `sendForm` handler). Keep them in sync if modified.

---

## What to Reference

- **`PRD.md`**: Complete specification of features, data models, API contracts, validation rules, email distribution logic, and cost calculation formulas. Consult this before making changes to ensure you understand the intended behavior.
- **`README.md`**: Basic installation instructions.
- **`config.json`**: Required secrets file (gitignored). Keys: `apikey`, `mailpass`, `otpkey`, `key`, `cert`.

---

## Common Pitfalls

- **Changing validation patterns**: The regex patterns in `server.js` and the `charPattern` attributes in `index.html` must stay in sync. A mismatch means the client allows characters the server rejects (or vice versa).
- **Socket.IO event names**: Both client and server reference event names as strings (`'getEvent'`, `'sendForm'`, `'ack'`). Renaming one side without the other silently breaks communication.
- **The `ack` convention**: When the third argument to `ack()` is a string, it's treated as an error. Any non-string value (including `undefined` or an object like the event data) is a success. On success, the data is forwarded to the client. Do not pass non-string truthy values as error indicators.
- **`EvLoad` flag**: Only one event can be loaded at a time. This is intentional to prevent concurrent Wild Apricot API calls.
- **`rData` on socket**: Receipt upload data is stored directly on the socket object (`sck.rData`). It is consumed and deleted on form submission.
- **Test email detection**: If the instructor email is `test@example.com`, the email subject is prefixed with `<<FORMBOT_TEST>>`. Do not remove this.

---

## Security Considerations

- **Never commit `config.json`** — it contains API keys and passwords.
- **Input validation is critical** — all user input flows into emails (HTML body and attachments). Maintain server-side validation even if client-side checks exist.
- **OTP tokens** protect against unauthorized access. Do not weaken or bypass the authentication flow.
- **File upload size limits** (`MaxUpload = 1GB`) prevent denial of service. Do not increase without consideration.
- **HTML in emails** is constructed by string concatenation. Be cautious about injecting user-supplied values into HTML without sanitization.
