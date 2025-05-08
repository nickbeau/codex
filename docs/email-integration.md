<!-- docs/email-integration.md -->
# Integrating Codex CLI via Email

This guide shows how to set up a simple email gateway that forwards incoming emails to the OpenAI Codex CLI and sends back its responses.

## Overview
1. Use an inbound-email provider (e.g. Mailgun, SendGrid, AWS SES) to parse incoming mail and POST its payload to your webhook.
2. Run a Node.js Express service that:
   - Receives `from`, `to`, `subject`, and `text` fields from the webhook.
   - Invokes the local `codex` CLI with your chosen approval mode.
   - Captures stdout (the agent’s reply).
   - Sends an email back to the original sender with the response.

## Prerequisites
- A publicly reachable server or container (Heroku, AWS, Docker, etc.)
- Node.js 18+ installed
- `codex` CLI installed globally (`npm install -g @openai/codex`)
- Environment vars set:
  ```bash
  export OPENAI_API_KEY="your-key"
  export SMTP_HOST="smtp.example.com"
  export SMTP_PORT=587
  export SMTP_USER="smtp-user"
  export SMTP_PASS="smtp-password"
  export SMTP_FROM="codex-bot@example.com"
  ```

## Example Code
Create a folder (e.g. `email-bot`) and add these two files:

### package.json
```json
{
  "name": "codex-email-bot",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "body-parser": "^1.20.1",
    "nodemailer": "^6.7.8"
  }
}
```

### server.js
```js
import express from 'express'
import bodyParser from 'body-parser'
import { spawn } from 'child_process'
import nodemailer from 'nodemailer'

const app = express()
app.use(bodyParser.urlencoded({ extended: true }))
app.use(bodyParser.json())

// Load SMTP settings from env
const { SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASS, SMTP_FROM } = process.env
if (!SMTP_HOST || !SMTP_USER || !SMTP_PASS || !SMTP_FROM) {
  console.error('SMTP configuration missing')
  process.exit(1)
}

// Set up mail transporter
const transporter = nodemailer.createTransport({
  host: SMTP_HOST,
  port: Number(SMTP_PORT) || 587,
  secure: SMTP_PORT == 465,
  auth: { user: SMTP_USER, pass: SMTP_PASS }
})

// Email webhook endpoint
app.post('/email', (req, res) => {
  const { from, subject, text } = req.body
  if (!from || !text) return res.status(400).send('Missing fields')

  // Invoke Codex CLI
  const codex = spawn('codex', ['--approval-mode', 'suggest', '--quiet', text.trim()], {
    env: { ...process.env, CODEX_QUIET_MODE: '1' }
  })
  let reply = '', errOut = ''
  codex.stdout.on('data', d => reply += d)
  codex.stderr.on('data', d => errOut += d)

  codex.on('close', async code => {
    let body = code === 0
      ? `Hello,\n\nCodex’s response:\n\n${reply}`
      : `Error (code ${code}):\n${errOut}`

    // Send back to the original sender
    await transporter.sendMail({
      from: SMTP_FROM,
      to: from,
      subject: `Re: ${subject || 'Codex request'}`,
      text: body
    })
    res.sendStatus(200)
  })
})

// Start server
const PORT = process.env.PORT || 3000
app.listen(PORT, () => {
  console.log(`Codex Email Bot listening on port ${PORT}`)
})
```

## Deployment Steps
1. `cd email-bot && npm install`
2. Ensure your SMTP env vars and `OPENAI_API_KEY` are set.
3. Deploy to your server and expose port 80/443.
4. Configure your email provider inbound webhook:
   - Point to `https://<your-host>/email`
   - Send form‐encoded or JSON with keys: `from`, `subject`, `text`.
5. Send an email to your inbox; the bot will reply via SMTP.

With this in place, any incoming email becomes a Codex prompt, and Codex’s answer is e-mailed back automatically.