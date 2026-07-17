# Create local project inside the vm you are utilizing for the cluster 
mkdir -p ~/pastebin-broken
cd ~/pastebin-broken

# Create package.json (BROKEN - missing comma on line 3)
cat > package.json <<'EOF'
{
  "name": "pastebin",
  "version": "1.0.0"
  "description": "Simple pastebin app",
  "main": "server.js",
  "dependencies": {
    "express": "^4.18.0",
    "body-parser": "^1.20.0"
  },
  "scripts": {
    "start": "node server.js"
  }
}
EOF

# Create server.js
cat > server.js <<'EOF'
const express = require('express');
const bodyParser = require('body-parser');
const app = express();
const PORT = 8080;

let pastes = [];

app.use(bodyParser.json());
app.use(express.static('views'));

app.get('/api/pastes', (req, res) => {
  res.json(pastes);
});

app.post('/api/paste', (req, res) => {
  const { text } = req.body;
  pastes.push({ id: pastes.length + 1, text });
  res.json({ success: true, id: pastes.length });
});

app.listen(PORT, () => {
  console.log(`Pastebin running on port ${PORT}`);
});
EOF

# Create views directory and HTML
mkdir views
cat > views/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
  <title>Pastebin</title>
  <style>
    body { font-family: sans-serif; max-width: 600px; margin: 50px auto; }
    textarea { width: 100%; height: 150px; }
    button { padding: 10px 20px; margin-top: 10px; }
    .paste { border: 1px solid #ccc; padding: 10px; margin: 10px 0; }
  </style>
</head>
<body>
  <h1>Pastebin</h1>
  <textarea id="pasteText" placeholder="Enter your paste..."></textarea>
  <button onclick="submit()">Submit</button>
  <div id="pastes"></div>
  
  <script>
    function submit() {
      const text = document.getElementById('pasteText').value;
      fetch('/api/paste', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ text })
      }).then(() => {
        document.getElementById('pasteText').value = '';
        loadPastes();
      });
    }
    
    function loadPastes() {
      fetch('/api/pastes')
        .then(r => r.json())
        .then(data => {
          document.getElementById('pastes').innerHTML = data.map(p => 
            `<div class="paste"><strong>#${p.id}:</strong> ${p.text}</div>`
          ).join('');
        });
    }
    
    loadPastes();
  </script>
</body>
</html>
EOF
