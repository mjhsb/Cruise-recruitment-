# Cruise-recruitment-
const express = require('express');
const path = require('path');
const fs = require('fs');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const cors = require('cors');

const PORT = process.env.PORT || 3000;
const DATA_DIR = path.join(__dirname, 'data');
const SUBMISSIONS_FILE = path.join(DATA_DIR, 'submissions.jsonl');

if (!fs.existsSync(DATA_DIR)) fs.mkdirSync(DATA_DIR, { recursive: true });

const app = express();

// Security middlewares
app.use(helmet());
app.use(cors({ origin: true })); // adjust origin in production to your domain
app.use(express.json({ limit: '10kb' }));

// Basic rate limiter for /apply to reduce spam
const applyLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 10, // limit each IP to 10 requests per windowMs
  standardHeaders: true,
  legacyHeaders: false,
  message: { message: 'Too many requests, please try again later.' }
});

app.use('/apply', applyLimiter);

// Serve static site from /public
app.use(express.static(path.join(__dirname, 'public'), { extensions: ['html'] }));

// Health check
app.get('/healthz', (req, res) => res.send({ status: 'ok' }));

// POST /apply endpoint
app.post('/apply', (req, res) => {
  const { fullName, email, phone, position, message, consent, submittedAt } = req.body || {};

  // Server-side validation
  if (!fullName || typeof fullName !== 'string' || fullName.trim().length < 2) {
    return res.status(400).json({ message: 'Invalid fullName' });
  }
  if (!email || typeof email !== 'string' || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    return res.status(400).json({ message: 'Invalid email' });
  }
  if (!phone || typeof phone !== 'string' || phone.trim().length < 7) {
    return res.status(400).json({ message: 'Invalid phone number' });
  }
  if (!position || typeof position !== 'string' || position.trim().length === 0) {
    return res.status(400).json({ message: 'Position is required' });
  }
  if (consent !== true && consent !== 'true') {
    return res.status(400).json({ message: 'Consent is required' });
  }

  const record = {
    id: Date.now() + '-' + Math.random().toString(36).slice(2, 9),
    fullName: fullName.trim(),
    email: email.trim(),
    phone: phone.trim(),
    position: position.trim(),
    message: message ? String(message).trim() : '',
    consent: true,
    ip: req.ip,
    userAgent: req.get('User-Agent') || '',
    submittedAt: submittedAt || new Date().toISOString()
  };

  // Append as JSON Lines (safe simple store) — in production use DB
  fs.appendFile(SUBMISSIONS_FILE, JSON.stringify(record) + '\n', (err) => {
    if (err) {
      console.error('Failed to save submission', err);
      return res.status(500).json({ message: 'Failed to save submission' });
    }

    // TODO: enqueue email/notification/job for recruiters, or forward to CRM
    res.status(200).json({ message: 'Application received', id: record.id });
  });
});

// Fallback for SPA/static site
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

app.listen(PORT, () => {
  console.log(`Server started on port ${PORT}`);
});
