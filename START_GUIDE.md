# 🚀 Yomnai Complete Startup Guide

## 📋 Prerequisites Check

Before starting, ensure you have these installed:

### Required Software
- ✅ **Python 3.10+** - Check: `python --version`
- ✅ **Node.js 18+** - Check: `node --version`
- ✅ **Ollama** - Check: `ollama --version`

### Install Missing Prerequisites

#### Windows
```powershell
# Install Python (if missing)
winget install Python.Python.3.11

# Install Node.js (if missing)
winget install OpenJS.NodeJS

# Install Ollama (if missing)
# Download from: https://ollama.ai/download/windows
```

#### WSL/Linux
```bash
# Install Python
sudo apt update
sudo apt install python3 python3-pip python3-venv

# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install Ollama
curl https://ollama.ai/install.sh | sh
```

---

## 🎯 Quick Start (One Command)

### Windows (PowerShell/Command Prompt)
```powershell
# Navigate to project directory
cd "C:\Users\computer\Documents\Bank Statement"

# Run the integrated startup script
python start_integrated.py
```

This single command will:
1. ✅ Check all requirements
2. ✅ Install dependencies automatically
3. ✅ Initialize the database
4. ✅ Add test email (test@example.com)
5. ✅ Start backend server
6. ✅ Start frontend server
7. ✅ Open browser automatically

**That's it! The app will be running at http://localhost:5173**

---

## 📝 Manual Start (Step-by-Step)

If you prefer to start services manually:

### Step 1: Start Ollama
```bash
# Terminal 1 - Start Ollama service
ollama serve

# Terminal 2 - Pull required models (first time only)
ollama pull qwen3-14b-32k:latest
ollama pull gemma3:4b
ollama pull qwen2.5:7b-vl
```

### Step 2: Setup Backend
```bash
# Terminal 3 - Navigate to backend
cd yomnai_backend

# Create virtual environment (first time only)
python -m venv ../venv

# Activate virtual environment
# Windows:
..\venv\Scripts\activate
# Linux/Mac:
source ../venv/bin/activate

# Install dependencies (first time only)
pip install -r requirements.txt

# Initialize database (first time only)
python init_database.py

# Add test email (first time only)
python admin_cli.py add-email test@example.com

# Start backend server
python run.py
```

Backend will run at: **http://localhost:8001**

### Step 3: Setup Frontend
```bash
# Terminal 4 - Navigate to frontend
cd yomnai_frontend

# Install dependencies (first time only)
npm install

# Start frontend development server
npm run dev
```

Frontend will run at: **http://localhost:5173**

---

## 🔐 Login & Use

1. **Open Browser**: http://localhost:5173
2. **Login Screen**: Enter `test@example.com`
3. **Upload Document**:
   - Click upload area
   - Select CSV, Excel, or PDF file
   - Wait for processing
4. **Explore Features**:
   - 💬 **Chat**: Ask questions about your data
   - 📊 **Analysis**: Run AI agent analysis
   - 📈 **Trends**: View transaction trends
   - 📋 **Transactions**: Browse all transactions

---

## 🛠️ Common Issues & Solutions

### Issue 1: "Ollama is not running"
```bash
# Solution: Start Ollama first
ollama serve

# Verify it's running
curl http://localhost:11434/api/tags
```

### Issue 2: "Port 8000 already in use"
```bash
# Windows - Find and kill process
netstat -ano | findstr :8000
taskkill /PID <process_id> /F

# Linux/Mac
lsof -i :8000
kill -9 <process_id>
```

### Issue 3: "Module not found" errors
```bash
# Backend fix
cd yomnai_backend
pip install -r requirements.txt

# Frontend fix
cd yomnai_frontend
npm install
```

### Issue 4: "Access denied" on login
```bash
# Add your email to whitelist
cd yomnai_backend
python admin_cli.py add-email your-email@example.com
```

### Issue 5: "Cannot connect to backend"
```bash
# Check backend is running
curl http://localhost:8001/health

# Check CORS settings in .env
VITE_API_URL=http://localhost:8001
```

---

## 📊 Verify Everything is Working

### 1. Check Backend Health
```bash
curl http://localhost:8001/health
```

Expected response:
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "services": {
    "database": "connected",
    "ollama": "connected",
    "chromadb": "connected"
  }
}
```

### 2. Check Frontend
Open http://localhost:5173 - You should see the login page

### 3. Check API Documentation
Open http://localhost:8001/docs - Interactive API documentation

---

## 🔧 Configuration Files

### Backend Configuration
**File**: `yomnai_backend/.env`
```env
# Ollama Configuration
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL_MAIN=qwen3-14b-32k:latest

# Security
SECRET_KEY=your-secret-key-here
SESSION_TIMEOUT_HOURS=24

# Frontend URL (for CORS)
FRONTEND_URLS=http://localhost:5173
```

### Frontend Configuration
**File**: `yomnai_frontend/.env`
```env
# API Configuration
VITE_API_URL=http://localhost:8001

# Feature Flags
VITE_ENABLE_WEBSOCKET=true
VITE_MAX_FILE_SIZE_MB=50
```

---

## 👥 Managing Users

### Add New User
```bash
cd yomnai_backend
python admin_cli.py add-email newuser@example.com
```

### List All Users
```bash
python admin_cli.py list-emails
```

### Remove User
```bash
python admin_cli.py remove-email user@example.com
```

### Bulk Add Users (CSV)
```bash
python admin_cli.py bulk-add users.csv
```

---

## 📁 Test Files

You can test with these sample files:

1. **Transaction CSV**: Any bank statement CSV
2. **Financial Statement**: Balance sheet or P&L Excel file
3. **PDF Document**: Bank statement or financial report PDF

Sample data structure for CSV:
```csv
Date,Description,Debit,Credit,Balance
2024-01-15,"SALARY TRANSFER",,15000,50000
2024-01-16,"ATM WITHDRAWAL",500,,49500
2024-01-17,"GROCERY STORE",350,,49150
```

---

## 🚦 Service Status Indicators

When running, you should see:

### Backend (Terminal)
```
INFO:     Uvicorn running on http://0.0.0.0:8000
INFO:     Application startup complete
INFO:     Database initialized successfully
INFO:     Agent service initialized successfully
```

### Frontend (Terminal)
```
VITE v5.0.0  ready in 500 ms
➜  Local:   http://localhost:5173/
➜  Network: use --host to expose
```

### Ollama (Terminal)
```
Ollama is running on http://localhost:11434
```

---

## 🔄 Stopping Services

### If using integrated script:
Press `Ctrl+C` in the terminal - it will gracefully stop all services

### If running manually:
1. Frontend: `Ctrl+C` in frontend terminal
2. Backend: `Ctrl+C` in backend terminal
3. Ollama: `Ctrl+C` in Ollama terminal

---

## 📱 Browser Requirements

Works best with:
- ✅ Chrome 90+
- ✅ Firefox 88+
- ✅ Edge 90+
- ✅ Safari 14+

---

## 🎉 Success Checklist

After starting, verify:
- [ ] Can access http://localhost:5173
- [ ] Can login with test@example.com
- [ ] Can upload a file
- [ ] Upload shows progress bar
- [ ] Can see parsed data
- [ ] Chat responds to questions
- [ ] Analysis runs successfully
- [ ] WebSocket shows "Connected"

---

## 📚 Next Steps

1. **Upload Real Data**: Use your actual bank statements
2. **Run Full Analysis**: Click "Start Analysis" to run all 12 agents
3. **Ask Questions**: Use chat to ask about your finances
4. **Explore Dashboards**: Navigate through different tabs
5. **Export Reports**: Download analysis results

---

## 💡 Pro Tips

1. **Faster Startup**: Use `python start_integrated.py` - it handles everything
2. **Better Performance**: Close other applications to free up RAM for AI models
3. **Persistent Data**: Your uploads are saved in `yomnai_backend/data/`
4. **Logs**: Check `yomnai_backend/logs/` for debugging
5. **API Testing**: Use http://localhost:8001/docs for interactive API testing

---

**Need Help?**
- Check logs: `yomnai_backend/logs/app.log`
- API Docs: http://localhost:8001/docs
- Review: `INTEGRATION_README.md`