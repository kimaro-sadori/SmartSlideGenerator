# Development Guide

## What We've Built So Far

You now have the **basic backend structure**. Here's what each file does:

### 📋 Files Explained

| File | Purpose |
|------|---------|
| `SmartSlideGenerator.Api.csproj` | Tells .NET which packages to download (like npm's package.json) |
| `Program.cs` | **The entry point** - where your app starts running |
| `appsettings.json` | **Configuration file** - stores API keys, settings |
| `.gitignore` | Tells Git what files NOT to save to GitHub |
| `HealthController.cs` | Your **first API endpoint** - test if API is working |

## 🚀 How to Run It Locally

### Step 1: Install .NET 8
Download from: https://dotnet.microsoft.com/download

### Step 2: Navigate to API folder
```bash
cd SmartSlideGenerator.Api
```

### Step 3: Restore dependencies
```bash
dotnet restore
```
This downloads all required packages.

### Step 4: Run the API
```bash
dotnet run
```

You should see:
```
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: https://localhost:5001
      Now listening on: http://localhost:5000
```

### Step 5: Test it!
Open your browser and go to:
```
http://localhost:5000/api/health
```

You should see:
```json
{
  "status": "API is running!",
  "timestamp": "2026-07-10T10:51:00Z"
}
```

## 📝 What You Learned

1. **C# Project Structure** - How .NET projects are organized
2. **Program.cs** - Where configuration happens
3. **Controllers** - How to create API endpoints
4. **Configuration** - How to store settings in appsettings.json

## 🎯 Next Steps

Once you verify the API is running, we'll add:

1. **Presentation Controller** - Main endpoint to generate slides
2. **Models** - Data structures for presentations
3. **Services** - Business logic for slide generation
4. **Kimi AI Integration** - Connect to AI API

## 💡 Tips

- **Swagger UI**: When running, visit `http://localhost:5000/swagger` to see all API endpoints
- **Hot Reload**: Press `Ctrl+C` to stop, then `dotnet run` to restart
- **Logs**: Watch the console for error messages

## ⚠️ Troubleshooting

**Port 5000 already in use?**
```bash
dotnet run --urls "http://localhost:5001"
```

**Dependencies won't install?**
```bash
dotnet clean
dotnet restore
```

---

**Ready to test it? Follow the steps above and let me know when you see the API response!** ✅
