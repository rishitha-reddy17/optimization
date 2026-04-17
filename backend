const express = require('express');
const cors = require('cors');
const crypto = require('crypto');

const app = express();
app.use(cors());
app.use(express.json());

// ─── In-Memory Store ─────────────────────────────────────────────────────────
const users = {
  admin: {
    password: crypto.createHash('sha256').update('Admin@1234').digest('hex'),
    role: 'admin',
    azure: null
  }
};
const sessions = {};

// ─── Helper ───────────────────────────────────────────────────────────────────
function auth(req, res, next) {
  const token = req.headers['authorization']?.split(' ')[1];
  if (!token || !sessions[token]) return res.status(401).json({ error: 'Unauthorized' });
  req.user = sessions[token];
  next();
}

// ─── Auth Routes ──────────────────────────────────────────────────────────────
app.post('/api/register', (req, res) => {
  const { username, password } = req.body;
  if (!username || !password) return res.status(400).json({ error: 'Username and password required' });
  if (users[username]) return res.status(409).json({ error: 'Username already exists' });
  if (password.length < 8) return res.status(400).json({ error: 'Password must be at least 8 characters' });
  users[username] = {
    password: crypto.createHash('sha256').update(password).digest('hex'),
    role: 'user',
    azure: null
  };
  res.json({ message: 'User registered successfully' });
});

app.post('/api/login', (req, res) => {
  const { username, password } = req.body;
  const user = users[username];
  if (!user) return res.status(401).json({ error: 'Invalid credentials' });
  const hashed = crypto.createHash('sha256').update(password).digest('hex');
  if (user.password !== hashed) return res.status(401).json({ error: 'Invalid credentials' });
  const token = crypto.randomBytes(32).toString('hex');
  sessions[token] = { username, role: user.role };
  res.json({ token, username, role: user.role });
});

app.post('/api/logout', auth, (req, res) => {
  const token = req.headers['authorization']?.split(' ')[1];
  delete sessions[token];
  res.json({ message: 'Logged out' });
});

// ─── Azure Connection Routes ───────────────────────────────────────────────────
app.post('/api/azure/connect', auth, async (req, res) => {
  const { tenantId, clientId, clientSecret, subscriptionId } = req.body;
  if (!tenantId || !clientId || !clientSecret || !subscriptionId)
    return res.status(400).json({ error: 'All Azure credentials are required' });

  try {
    // Attempt to get an Azure AD token
    const tokenUrl = `https://login.microsoftonline.com/${tenantId}/oauth2/v2.0/token`;
    const params = new URLSearchParams({
      grant_type: 'client_credentials',
      client_id: clientId,
      client_secret: clientSecret,
      scope: 'https://management.azure.com/.default'
    });

    const tokenRes = await fetch(tokenUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: params
    });
    const tokenData = await tokenRes.json();
    if (!tokenData.access_token) {
      return res.status(401).json({ error: tokenData.error_description || 'Azure authentication failed' });
    }

    // Store credentials per user
    users[req.user.username].azure = { tenantId, clientId, clientSecret, subscriptionId, accessToken: tokenData.access_token };
    res.json({ message: 'Azure connected successfully', subscriptionId });
  } catch (err) {
    res.status(500).json({ error: 'Failed to connect to Azure: ' + err.message });
  }
});

app.get('/api/azure/status', auth, (req, res) => {
  const azure = users[req.user.username]?.azure;
  res.json({ connected: !!azure, subscriptionId: azure?.subscriptionId || null });
});

// ─── Cost Routes ───────────────────────────────────────────────────────────────
app.get('/api/costs', auth, async (req, res) => {
  const azure = users[req.user.username]?.azure;
  if (!azure) return res.status(400).json({ error: 'Azure not connected' });

  try {
    // Refresh token
    const tokenUrl = `https://login.microsoftonline.com/${azure.tenantId}/oauth2/v2.0/token`;
    const params = new URLSearchParams({
      grant_type: 'client_credentials',
      client_id: azure.clientId,
      client_secret: azure.clientSecret,
      scope: 'https://management.azure.com/.default'
    });
    const tokenRes = await fetch(tokenUrl, { method: 'POST', headers: { 'Content-Type': 'application/x-www-form-urlencoded' }, body: params });
    const tokenData = await tokenRes.json();
    if (!tokenData.access_token) return res.status(401).json({ error: 'Azure token refresh failed' });

    const token = tokenData.access_token;
    const subId = azure.subscriptionId;

    // Get current month range
    const now = new Date();
    const from = new Date(now.getFullYear(), now.getMonth(), 1).toISOString().split('T')[0];
    const to = now.toISOString().split('T')[0];

    const costUrl = `https://management.azure.com/subscriptions/${subId}/providers/Microsoft.CostManagement/query?api-version=2023-11-01`;
    const body = {
      type: 'ActualCost',
      timeframe: 'Custom',
      timePeriod: { from, to },
      dataset: {
        granularity: 'Daily',
        aggregation: { totalCost: { name: 'Cost', function: 'Sum' } },
        grouping: [{ type: 'Dimension', name: 'ServiceName' }]
      }
    };

    const [actualRes, amortizedRes] = await Promise.all([
      fetch(costUrl, { method: 'POST', headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' }, body: JSON.stringify(body) }),
      fetch(costUrl, { method: 'POST', headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' }, body: JSON.stringify({ ...body, type: 'AmortizedCost' }) })
    ]);

    const actualData = await actualRes.json();
    const amortizedData = await amortizedRes.json();

    if (actualData.error) return res.status(400).json({ error: actualData.error.message });

    // Process rows
    const parseRows = (data) => {
      const rows = data?.properties?.rows || [];
      const cols = data?.properties?.columns || [];
      const costIdx = cols.findIndex(c => c.name === 'Cost');
      const svcIdx = cols.findIndex(c => c.name === 'ServiceName');
      const dateIdx = cols.findIndex(c => c.name === 'UsageDate');
      const map = {};
      rows.forEach(row => {
        const svc = row[svcIdx] || 'Unknown';
        if (!map[svc]) map[svc] = { service: svc, cost: 0, daily: [] };
        map[svc].cost += parseFloat(row[costIdx] || 0);
        map[svc].daily.push({ date: String(row[dateIdx]), cost: parseFloat(row[costIdx] || 0) });
      });
      return Object.values(map).sort((a, b) => b.cost - a.cost);
    };

    const actual = parseRows(actualData);
    const amortized = parseRows(amortizedData);
    const totalActual = actual.reduce((s, r) => s + r.cost, 0);
    const totalAmortized = amortized.reduce((s, r) => s + r.cost, 0);

    res.json({ actual, amortized, totalActual, totalAmortized, period: { from, to } });
  } catch (err) {
    res.status(500).json({ error: 'Failed to fetch costs: ' + err.message });
  }
});

// ─── Start ─────────────────────────────────────────────────────────────────────
const PORT = process.env.PORT || 4000;
app.listen(PORT, () => console.log(`Azure Cost Dashboard API running on http://localhost:${PORT}`));
module.exports = app;
