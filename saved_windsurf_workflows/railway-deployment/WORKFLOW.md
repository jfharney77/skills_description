---
description: How to deploy the influencer dashboard to Railway
---

# Railway Deployment Guide

## Prerequisites
- GitHub repository with the code
- Railway account
- Railway CLI (optional, but recommended)

## Service Configuration

### Backend Service
- **Root Directory:** `/backend`
- **Internal Port:** 8080
- **Dockerfile:** Uses Python 3.12 with uvicorn
- **Environment Variables:** None required

### Frontend Service
- **Root Directory:** `/frontend`
- **Internal Port:** 3000
- **Dockerfile:** Uses Node.js 22 with Vite and serve
- **Environment Variables:**
  - `VITE_API_BASE_URL`: Backend service URL (see below)

## Important: VITE_API_BASE_URL Configuration

The frontend requires `VITE_API_BASE_URL` to be set correctly in the Railway Variables tab.

**Format:** `https://your-backend.up.railway.app/api`

**Requirements:**
- Must include `https://` prefix
- Must include `/api` at the end (backend routes are under `/api/`)
- No port number
- No trailing slash after `/api`

**Example:**
```
VITE_API_BASE_URL=https://windsurftutorials-production.up.railway.app/api
```

**Critical Notes:**
- Vite bakes environment variables at build time
- After setting `VITE_API_BASE_URL`, you MUST trigger a manual redeploy from the Deployments tab
- If you update the variable after a build, it won't take effect until you redeploy

## Common Issues and Solutions

### Issue: Node.js Version Error
**Error:** `Vite requires Node.js version 20.19+ or 22.12+`

**Solution:** Update frontend Dockerfile to use `node:22-alpine` instead of `node:18-alpine`

### Issue: "SyntaxError: Unexpected token '<', "<!doctype "... is not valid JSON"
**Cause:** Frontend receiving HTML instead of JSON from API

**Solutions:**
1. Ensure `VITE_API_BASE_URL` is set in Railway Variables tab
2. Verify format: `https://backend-url.up.railway.app/api`
3. Check that `https://` prefix is included
4. Verify `/api` suffix is included
5. Trigger manual redeploy after setting the variable

### Issue: 404 Not Found on API endpoints
**Cause:** Missing `/api` prefix in URL

**Solution:** Ensure `VITE_API_BASE_URL` includes `/api` at the end

## Deployment Steps

1. **Create Backend Service**
   - New Project → Deploy from GitHub
   - Select repository
   - Set Root Directory to `/backend`
   - Set Internal Port to `8080`
   - Deploy

2. **Create Frontend Service**
   - Add New Service → Deploy from GitHub
   - Select same repository
   - Set Root Directory to `/frontend`
   - Set Internal Port to `3000`
   - Add Variable: `VITE_API_BASE_URL` = `https://your-backend-url.up.railway.app/api`
   - Deploy

3. **Generate Public Domains**
   - Railway will automatically generate public domains for both services
   - Note the backend URL for frontend configuration

4. **Configure Frontend Environment Variable**
   - Go to Frontend Service → Variables tab
   - Set `VITE_API_BASE_URL` to backend URL with `/api` suffix
   - Go to Deployments tab → Click "Redeploy" button

5. **Verify Deployment**
   - Access frontend public URL
   - Select an influencer from dropdown
   - Verify tweets load with sentiment badges

## Files Required for Railway Deployment

### Backend
- `Dockerfile` - Docker configuration
- `railway.json` - Forces Dockerfile builder
- `requirements.txt` - Python dependencies
- `.env.example` - Environment variable template

### Frontend
- `Dockerfile` - Docker configuration with VITE_API_BASE_URL build arg
- `railway.json` - Forces Dockerfile builder
- `package.json` - Node dependencies
- `.env.example` - Environment variable template
