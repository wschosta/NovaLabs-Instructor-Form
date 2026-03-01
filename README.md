# NovaLabs Instructor FormBot

This is a webapp written in Node.js which automates the process of filling out instructor Reimbursement Forms at NovaLabs.

## Installation

1. Install Node.js and NPM
2. In the project directory, run `npm i` to install dependencies
3. Create a `config.json` file in the project root with the following keys:
   ```json
   {
     "apikey": "Wild Apricot API key",
     "mailpass": "Gmail app password for SMTP",
     "otpkey": "TOTP shared secret for client authentication",
     "key": "/path/to/ssl/private.key",
     "cert": "/path/to/ssl/certificate.crt"
   }
   ```
   The `key` and `cert` fields are optional — if omitted or invalid, the server falls back to HTTP.
4. Launch with `node server`
5. Default endpoint is `localhost:8080` (HTTPS if certs are configured, HTTP otherwise)

## Production Deployment

A systemd service file (`formbot.service`) and launch script (`run.sh`) are included. Update the `ExecStart` path in `formbot.service` to match your installation directory before enabling the service.

## Documentation

- **`PRD.md`** — Full product requirements: features, data models, API contracts, validation rules, email logic, and cost formulas.
- **`CLAUDE.md`** — Development guidelines: architecture constraints, code style, key technical details, and common pitfalls.

## License

GNU GPL v3
