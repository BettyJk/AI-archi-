# MG2 NOK Correction System Integration Guide

## Overview

This guide explains how to integrate the **MG2 NOK Correction System** with the **space-weaver** packaging analysis application. The integration enables AI-powered analysis of clearance violations directly from the 3D viewer's captured images.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ space-weaver (React/TypeScript - Cloudflare Workers)           │
│                                                                 │
│  - 3D Viewer (Viewer3D.tsx)                                   │
│  - Clearance Table (ClearanceTable.tsx)                       │
│  - "Analyze" Button → calls MG2 API                          │
│                                                                 │
│  Port: 3000/5173 (dev)                                        │
└─────────────────────────────────────────────────────────────────┘
           ↓ API calls (HTTP POST)
┌─────────────────────────────────────────────────────────────────┐
│ MG2 FastAPI Server (mg2_api_server.py)                         │
│                                                                 │
│  - Accepts: study_part, neighbour_part, image_base64          │
│  - Processes: image classification + AI analysis             │
│  - Returns: verdict, gap_mm, analysis details               │
│                                                                 │
│  Port: 8001                                                    │
└─────────────────────────────────────────────────────────────────┘
           ↓ Python pipeline calls
┌─────────────────────────────────────────────────────────────────┐
│ MG2 Pipeline (Python)                                          │
│                                                                 │
│  - classifier.py: Classify images, extract gap/parts         │
│  - ai_analyst.py: Run AI analysis on NOK images              │
│  - report.py: Generate HTML reports                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Installation

### 1. Install MG2 Dependencies

```bash
cd MG2_NOK_Correction_System_fixed
pip install -r requirements.txt
```

This installs:
- FastAPI & Uvicorn (for the API server)
- Anthropic (for AI analysis)
- Pillow (for image processing)
- All other existing dependencies

### 2. Environment Setup

Create a `.env` file in `MG2_NOK_Correction_System_fixed/` with:

```env
# AI API Configuration
ANTHROPIC_API_KEY=your_anthropic_api_key_here

# Optional: For Capgemini API
CAPGEMINI_ASSETS_API_KEY=your_capgemini_key_here

# MG2 Server Configuration
MG2_HOST=0.0.0.0
MG2_PORT=8001
DEBUG=false
```

**Required:**
- `ANTHROPIC_API_KEY`: Get from https://console.anthropic.com/

**Optional:**
- Use `CAPGEMINI_ASSETS_API_KEY` for Capgemini API instead

## Usage

### Starting the MG2 API Server

```bash
# From MG2_NOK_Correction_System_fixed directory
python mg2_api_server.py
```

Expected output:
```
╔══════════════════════════════════════════════════════════════════╗
║   MG2 NOK Correction System - FastAPI Server                   ║
║   Starting on 0.0.0.0:8001                                      ║
╚══════════════════════════════════════════════════════════════════╝

INFO:     Uvicorn running on http://0.0.0.0:8001
```

### Using in space-weaver

1. **Start space-weaver dev server:**
   ```bash
   cd space-weaver-main
   npm run dev
   ```

2. **Open the application:**
   - Navigate to http://localhost:5173 (or 3000)

3. **Perform Clearance Analysis:**
   - Load a zone
   - Select a study part
   - Select neighbouring parts
   - Click "3D-ENV" to open viewer
   - Click "CC" to compute clearance
   - Results appear in the Clearance Table

4. **Run AI Analysis:**
   - In the Clearance Table, for each row, click **"Analyze"** button
   - The system will:
     * Capture a multi-view AI composite from the 3D viewer (2400×1600)
     * Send the image to the MG2 API
     * Display the analysis verdict (CLASH, NOK, WARNING, OK)
     * Store analysis results

## API Reference

### Health Check

```
GET /health

Response:
{
  "status": "ok",
  "service": "MG2 NOK Correction System",
  "timestamp": "2026-06-06T10:30:00.000Z"
}
```

### Single Image Analysis

```
POST /analyze

Request:
{
  "study_part": "part_name",
  "neighbour_part": "neighbour_name",
  "image_base64": "base64_encoded_png_image"
}

Response:
{
  "study_part": "part_name",
  "neighbour_part": "neighbour_name",
  "gap_mm": 5.234,
  "verdict": "WARNING",
  "analysis": {
    "rule_applied": "CCP-02",
    "required_min_mm": 15,
    "status": "WARNING",
    "findings": [...]
  },
  "error": null
}
```

### Batch Analysis

```
POST /analyze-batch

Request:
{
  "images": [
    {
      "study_part": "part_1",
      "neighbour_part": "part_2",
      "image_base64": "..."
    },
    ...
  ],
  "api_type": "anthropic",
  "include_ok": false
}

Response:
{
  "analysis_id": "20260606_103000",
  "timestamp": "2026-06-06T10:30:00Z",
  "total_images": 2,
  "results": [...]
}
```

## File Changes Summary

### New Files Created

1. **`MG2_NOK_Correction_System_fixed/mg2_api_server.py`**
   - FastAPI server wrapper around MG2 pipeline
   - Provides REST API endpoints for image analysis
   - Handles base64 image uploads and processing

### Modified Files

1. **`space-weaver-main/src/lib/api.ts`**
   - Added `analyzeClearanceImage()` function
   - Added `analyzeClearanceBatch()` function
   - Added `checkMG2Service()` health check
   - New types: `MG2AnalysisRequest`, `MG2AnalysisResult`, `MG2BatchRequest`, `MG2BatchResponse`

2. **`space-weaver-main/src/components/ClearanceTable.tsx`**
   - Added `onAnalyze` callback prop
   - Added "Analyze" button in action row
   - Updated `CamBtn` component to support `accent` style prop
   - Added `analyzingRow` state tracking

3. **`space-weaver-main/src/routes/index.tsx`**
   - Imported `analyzeClearanceImage` from api.ts
   - Added `analysisResults` state
   - Added `handleAnalyzeImage()` handler function
   - Passed `onAnalyze` to ClearanceTable component

4. **`MG2_NOK_Correction_System_fixed/requirements.txt`**
   - Added `fastapi>=0.100.0`
   - Added `uvicorn>=0.23.0`

## Troubleshooting

### MG2 Server Won't Start

```bash
# Check if port 8001 is in use
lsof -i :8001  # macOS/Linux
netstat -ano | findstr :8001  # Windows

# Try a different port
MG2_PORT=8002 python mg2_api_server.py
```

### "Cannot reach MG2 service" Error

1. Verify MG2 server is running on port 8001
2. Check firewall settings
3. Verify API_BASE in `lib/api.ts` matches server URL
4. Check browser console for CORS errors

### CORS Issues

The MG2 server accepts requests from:
- `http://localhost:3000`
- `http://localhost:5173`
- `*` (all origins) - can be restricted in production

### Image Capture Fails

- Ensure 3D-ENV viewer is open with meshes loaded
- Check that neighbour has valid segment data (point_a and point_b)
- Try resizing viewer window or closing other applications

### Analysis Returns No Results

1. Check ANTHROPIC_API_KEY is set correctly
2. Verify API key has sufficient credits
3. Check MG2 server logs for errors
4. Try a simpler test image first

## Development

### Testing the API Directly

```bash
# Health check
curl http://localhost:8001/health

# Test analysis with a sample image
curl -X POST http://localhost:8001/analyze \
  -H "Content-Type: application/json" \
  -d '{
    "study_part": "test_part",
    "neighbour_part": "test_neighbour",
    "image_base64": "iVBORw0KGgoAAAANS..."
  }'
```

### Debugging

Enable debug mode:
```bash
DEBUG=true python mg2_api_server.py
```

This enables Uvicorn auto-reload and more verbose logging.

### Monitoring

The MG2 server logs include:
- Request/response times
- Analysis verdicts
- API call details
- Error messages

## Future Enhancements

Possible improvements for future versions:

1. **Batch Operations**
   - Analyze all rows at once
   - Download aggregated report

2. **Result Display**
   - Show AI analysis details in UI modal
   - Display extracted rules and findings

3. **Report Generation**
   - Generate HTML reports from batch analysis
   - Export results as PDF

4. **Caching**
   - Cache analysis results to avoid re-processing
   - Store analysis history

5. **Integration with Backend**
   - Store analysis results in database
   - Link results to clearance calculation history

## Support

For issues or questions:

1. Check the troubleshooting section above
2. Review MG2 server logs
3. Verify all dependencies are installed: `pip list`
4. Check .env file for correct API keys
5. Ensure ports 8001 (MG2) and 5173/3000 (space-weaver) are available
