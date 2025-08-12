# Uc-ePIN
UC al
const express = require('express');
const cors = require('cors');
const sqlite3 = require('sqlite3').verbose();
const bodyParser = require('body-parser');

const app = express();
const PORT = 4000;

app.use(cors());
app.use(bodyParser.json());

// DB setup
const db = new sqlite3.Database('./db.sqlite');

db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    pubg_uid TEXT UNIQUE,
    last_claim INTEGER DEFAULT 0,
    balance INTEGER DEFAULT 0
  )`);
  db.run(`CREATE TABLE IF NOT EXISTS claims (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,
    amount INTEGER,
    claimed_at INTEGER
  )`);
});

// Middleware to check cooldown
function checkCooldown(req, res, next) {
  const { pubg_uid } = req.body;
  if (!pubg_uid) return res.status(400).json({ error: 'PUBG UID required' });

  db.get(`SELECT last_claim FROM users WHERE pubg_uid = ?`, [pubg_uid], (err, row) => {
    if (err) return res.status(500).json({ error: err.message });
    const now = Date.now();
    if (row) {
      if (now - row.last_claim < 10 * 60 * 1000) {
        const wait = Math.ceil((10 * 60 * 1000 - (now - row.last_claim)) / 1000);
        return res.status(429).json({ error: `Wait ${wait} seconds before next claim.` });
      }
    }
    next();
  });
}

// Claim endpoint
app.post('/api/claim', checkCooldown, (req, res) => {
  const { pubg_uid } = req.body;
  const now = Date.now();
  const amount = 60; // 60 UC per claim

  db.serialize(() => {
    db.get(`SELECT id, balance FROM users WHERE pubg_uid = ?`, [pubg_uid], (err, row) => {
      if (err) return res.status(500).json({ error: err.message });
      if (row) {
        // Update user balance and last_claim
        const newBalance = row.balance + amount;
        db.run(
          `UPDATE users SET balance = ?, last_claim = ? WHERE id = ?`,
          [newBalance, now, row.id],
          function (err) {
            if (err) return res.status(500).json({ error: err.message });
            db.run(
              `INSERT INTO claims (user_id, amount, claimed_at) VALUES (?, ?, ?)`,
              [row.id, amount, now]
            );
            res.json({ success: true, balance: newBalance });
          }
        );
      } else {
        // New user
        db.run(
          `INSERT INTO users (pubg_uid, last_claim, balance) VALUES (?, ?, ?)`,
          [pubg_uid, now, amount],
          function (err) {
            if (err) return res.status(500).json({ error: err.message });
            const userId = this.lastID;
            db.run(
              `INSERT INTO claims (user_id, amount, claimed_at) VALUES (?, ?, ?)`,
              [userId, amount, now]
            );
            res.json({ success: true, balance: amount });
          }
        );
      }
    });
  });
});

// Get user balance
app.get('/api/balance/:pubg_uid', (req, res) => {
  const pubg_uid = req.params.pubg_uid;
  db.get(`SELECT balance FROM users WHERE pubg_uid = ?`, [pubg_uid], (err, row) => {
    if (err) return res.status(500).json({ error: err.message });
    if (!row) return res.json({ balance: 0 });
    res.json({ balance: row.balance });
  });
});

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});





import React, { useState, useEffect } from 'react';

function App() {
  const [pubgUID, setPubgUID] = useState('');
  const [balance, setBalance] = useState(null);
  const [message, setMessage] = useState('');
  const [loading, setLoading] = useState(false);
  const [showAd, setShowAd] = useState(false);
  const [adTime, setAdTime] = useState(10);

  useEffect(() => {
    let timer;
    if (showAd && adTime > 0) {
      timer = setTimeout(() => setAdTime(adTime - 1), 1000);
    } else if (adTime === 0) {
      setShowAd(false);
      setAdTime(10);
      claimUC();
    }
    return () => clearTimeout(timer);
  }, [showAd, adTime]);

  const claimUC = () => {
    setLoading(true);
    fetch('http://localhost:4000/api/claim', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ pubg_uid: pubgUID }),
    })
      .then((res) => res.json())
      .then((data) => {
        setLoading(false);
        if (data.success) {
          setMessage(`Uğurla +60 UC əlavə olundu! Balansınız: ${data.balance}`);
          setBalance(data.balance);
        } else if (data.error) {
          setMessage(`Xəta: ${data.error}`);
        }
      })
      .catch(() => {
        setLoading(false);
        setMessage('Serverlə əlaqə problemi.');
      });
  };

  const startClaim = () => {
    if (!pubgUID.trim()) {
      setMessage('Zəhmət olmasa PUBG UID daxil edin.');
      return;
    }
    setMessage('');
    setShowAd(true);
  };

  const checkBalance = () => {
    if (!pubgUID.trim()) {
      setMessage('Zəhmət olmasa PUBG UID daxil edin.');
      return;
    }
    fetch(`http://localhost:4000/api/balance/${pubgUID}`)
      .then((res) => res.json())
      .then((data) => {
        setBalance(data.balance);
        setMessage(`Sizin balansınız: ${data.balance} UC`);
      });
  };

  return (
    <div style={{ maxWidth: 400, margin: 'auto', padding: 20, fontFamily: 'Arial' }}>
      <h2>PUBG UC Claim Sistemi</h2>
      <input
        type="text"
        placeholder="PUBG UID daxil edin"
        value={pubgUID}
        onChange={(e) => setPubgUID(e.target.value)}
        style={{ width: '100%', padding: 8, marginBottom: 10 }}
      />
      <button onClick={startClaim} disabled={loading || showAd} style={{ width: '100%', padding: 10 }}>
        {loading ? 'İşləyir...' : 'UC Claim Et'}
      </button>
      <button onClick={checkBalance} disabled={loading || showAd} style={{ width: '100%', padding: 10, marginTop: 10 }}>
        Balansı Yoxla
      </button>
      {message && <p style={{ marginTop: 20 }}>{message}</p>}
      {showAd && (
        <div style={{ marginTop: 20, padding: 20, border: '2px solid #333', textAlign: 'center' }}>
          <p>Reklam - zəhmət olmasa {adTime} saniyə gözləyin...</p>
        </div>
      )}
    </div>
  );
}

export default App;



npm init -y
npm install express cors sqlite3 body-parser
node server.js


npx create-react-app pubg-uc-claim
cd pubg-uc-claim
# src/App.js-i yuxarıdakı kodla əvəzlə
npm start
