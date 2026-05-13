---
description: How to fix CORS (Cross-Origin Resource Sharing) errors in web applications
---

# Fixing CORS Errors

CORS errors occur when a web application tries to make requests to a different origin (domain, port, or protocol) than the one that served the page. Here's how to fix them:

## For FastAPI (Python)

Add CORS middleware to your FastAPI application:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Enable CORS for development
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # For development only - use specific origins in production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**Important:** For production, replace `allow_origins=["*"]` with specific origins:
```python
allow_origins=["https://yourdomain.com", "https://app.yourdomain.com"]
```

## For Express (Node.js)

Install and configure cors:
```bash
npm install cors
```

```javascript
const express = require('express');
const cors = require('cors');
const app = express();

// For development
app.use(cors());

// For production with specific origins
app.use(cors({
  origin: ['https://yourdomain.com', 'https://app.yourdomain.com']
}));
```

## For Flask (Python)

Install and configure flask-cors:
```bash
pip install flask-cors
```

```python
from flask import Flask
from flask_cors import CORS

app = Flask(__name__)

# For development
CORS(app)

# For production with specific origins
CORS(app, resources={r"/api/*": {"origins": ["https://yourdomain.com"]}})
```

## Common CORS Issues

1. **Browser preview ports**: When using browser previews (like Windsurf's), the port may differ from localhost:5173. Use `allow_origins=["*"]` for development.

2. **Preflight requests**: Ensure your server handles OPTIONS requests. Most CORS middleware does this automatically.

3. **Credentials**: If using cookies or authentication, set `allow_credentials=True` and specify exact origins (not wildcards).

4. **Backend restart required**: After changing CORS settings, you must restart your backend server for changes to take effect.

## Debugging

- Check browser console for specific CORS error messages
- Verify the backend is running and accessible
- Ensure the API URL in frontend matches the backend URL
- Check if proxy configuration in Vite/webpack is working correctly
