# Integration Complete — Quick Start Guide

## What Was Integrated

The **MG2 NOK Correction System** has been successfully integrated into the **space-weaver** packaging analysis app. When you click "Analyze" on a clearance result, the system will:

1. ✓ Capture the CamViewAI multi-view composite image from the 3D viewer
2. ✓ Send it to the MG2 FastAPI server on port 8001
3. ✓ Run AI analysis to determine verdict (CLASH, NOK, WARNING, OK)
4. ✓ Return analysis results with extracted rules and findings

## Files Modified/Created

### New Files
- `MG2_NOK_Correction_System_fixed/mg2_api_server.py` — FastAPI server wrapper
- `INTEGRATION_GUIDE.md` — Comprehensive integration documentation
- `setup_integration.sh` — Linux/macOS setup script
- `setup_integration.bat` — Windows setup script
- `QUICK_START.md` — This file

### Modified Files
- `space-weaver-main/src/lib/api.ts` — Added MG2 API functions
- `space-weaver-main/src/components/ClearanceTable.tsx` — Added "Analyze" button
- `space-weaver-main/src/routes/index.tsx` — Added analysis handler
- `MG2_NOK_Correction_System_fixed/requirements.txt` — Added FastAPI dependencies

## Quick Start (5 Minutes)

### Windows
```bash
# 1. Run setup script
setup_integration.bat

# 2. Edit .env file and add API key
notepad MG2_NOK_Correction_System_fixed\.env

# 3. Terminal 1: Start MG2 server
cd MG2_NOK_Correction_System_fixed
venv_mg2\Scripts\activate.bat
python mg2_api_server.py

# 4. Terminal 2: Start space-weaver
cd space-weaver-main
npm run dev

# 5. Open browser: http://localhost:5173
```

### Linux/macOS
```bash
# 1. Run setup script
bash setup_integration.sh

# 2. Edit .env file and add API key
vi MG2_NOK_Correction_System_fixed/.env

# 3. Terminal 1: Start MG2 server
source MG2_NOK_Correction_System_fixed/venv_mg2/bin/activate
python MG2_NOK_Correction_System_fixed/mg2_api_server.py

# 4. Terminal 2: Start space-weaver
cd space-weaver-main
npm run dev

# 5. Open browser: http://localhost:5173
```

## Using the AI Analysis Feature

1. **Load a Zone**
   - Select a zone (e.g., "Peugeot 3008 [...]")

2. **Set Study Part & Neighbours**
   - Click on a part (study part)
   - Select one or more neighbours

3. **View Clearance**
   - Click "3D-ENV" button to open 3D viewer
   - Click "CC" to compute mesh-to-mesh clearance

4. **Run AI Analysis**
   - In Clearance Table, click **"Analyze"** button for each row
   - Status shows: "Analyzing…" → Result appears
   - Verdict displayed: CLASH / NOK / WARNING / OK

## API Configuration

You need an **Anthropic API Key** to run AI analysis:

1. Visit https://console.anthropic.com/
2. Create account or sign in
3. Get your API key
4. Add to `.env` file:
   ```
   ANTHROPIC_API_KEY=sk-ant-...
   ```

## Troubleshooting

### Port Already in Use
```bash
# Check what's using port 8001
lsof -i :8001  # macOS/Linux
netstat -ano | findstr :8001  # Windows

# Use different port
MG2_PORT=8002 python mg2_api_server.py
```

### MG2 Server Not Starting
- Check Python version: `python --version` (needs 3.8+)
- Check dependencies: `pip list`
- Verify ANTHROPIC_API_KEY is set: `echo $ANTHROPIC_API_KEY`

### "Cannot reach MG2 service"
- Verify MG2 server is running: `curl http://localhost:8001/health`
- Check browser console for errors
- Verify CORS settings (should allow localhost:5173)

## Features

✅ **Capture from 3D Viewer** — Multi-view AI composite (2400×1600)
✅ **Real-time Analysis** — Send images directly to MG2
✅ **Verdict Classification** — CLASH, NOK, WARNING, OK
✅ **AI-Powered** — Uses Anthropic Claude for analysis
✅ **Batch Support** — API ready for batch analysis (coming soon)
✅ **Report Generation** — HTML reports (coming soon)

## Architecture

```
┌─────────────────┐
│  space-weaver   │
│   (React)       │
│                 │
│ [Analyze] ─────┐│
└────────────────┘│
                  │ HTTP POST
                  │ {image_base64}
                  ▼
         ┌─────────────────────┐
         │   MG2 API Server    │
         │   (FastAPI)         │
         │   Port: 8001        │
         │                     │
         │ ┌─────────────────┐ │
         │ │  Pipeline:      │ │
         │ │ • Classifier    │ │
         │ │ • AI Analyst    │ │
         │ │ • Report Gen    │ │
         │ └─────────────────┘ │
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────┐
         │  Anthropic API  │
         │  (Claude)       │
         └─────────────────┘
```

## Next Steps

1. ✅ Complete: Basic AI Analysis integration
2. ⏳ Coming Soon: 
   - Analysis results display in UI
   - Batch analysis with report download
   - Result storage and history
   - Advanced filtering by verdict

## Support

For detailed information, see: **INTEGRATION_GUIDE.md**

For API reference: **INTEGRATION_GUIDE.md** → "API Reference" section

Questions? Check the troubleshooting section above first.

---

**Integration Status: ✅ Complete**

Last Updated: 2026-06-06
