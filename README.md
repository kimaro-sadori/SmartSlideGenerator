# SmartSlideGenerator - Complete Development Guide

> **Your AI-Powered PowerPoint Generator Built Step-by-Step**

This is a comprehensive guide documenting every file, every line of code, and why it exists. Use this as your reference when debugging or learning.

## 📖 Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture Explained](#architecture-explained)
3. [File-by-File Breakdown](#file-by-file-breakdown)
4. [Step-by-Step Setup](#step-by-step-setup)
5. [How Everything Works](#how-everything-works)
6. [Common Errors & Solutions](#common-errors--solutions)
7. [Testing Guide](#testing-guide)

---

## 📋 Project Overview

### What We Built

A full-stack web application that:
1. Takes a topic from the user
2. Uses Kimi AI to generate presentation content
3. Structures the content into slides
4. Applies professional layouts
5. (Soon) Generates PowerPoint files
6. Returns the presentation to download

### Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Frontend** | SvelteKit + Svelte | Beautiful, reactive UI |
| **Backend** | C# .NET 8 | High-performance API |
| **AI** | Kimi API | Content generation |
| **Database** | PostgreSQL (future) | Data persistence |
| **DevOps** | Docker (future) | Containerization |

### Why This Stack?

- **C# .NET**: Type-safe, fast, enterprise-grade
- **SvelteKit**: Modern, lightweight, excellent DX
- **Kimi**: Non-US, data sovereign, free tier available
- **PostgreSQL**: Open-source, reliable, free

---

## 🏗️ Architecture Explained

### Clean Architecture Pattern

```
┌─────────────────────────────────────┐
│     User Interface (SvelteKit)      │
│  - Beautiful forms                  │
│  - Real-time feedback               │
│  - Error handling                   │
└────────────────┬────────────────────┘
                 │ HTTP / JSON
┌─────────────────▼────────────────────┐
│     API Controllers (.NET)            │
│  - Route HTTP requests               │
│  - Validate input                    │
│  - Call services                     │
└────────────────┬────────────────────┘
                 │ Dependency Injection
┌─────────────────▼────────────────────┐
│     Business Logic (Services)        │
│  - Generate slide content            │
│  - Apply layouts                     │
│  - Manage storage                    │
└────────────────┬────────────────────┘
                 │
┌─────────────────▼────────────────────┐
│     External Services                │
│  - Kimi AI                           │
│  - File Storage                      │
│  - Database (future)                 │
└─────────────────────────────────────┘
```

### Why This Matters

- **Separation of Concerns**: Each layer has one job
- **Testability**: Easy to test each piece separately
- **Reusability**: Services can be reused by multiple controllers
- **Maintainability**: Easy to find and fix bugs
- **Scalability**: Can swap implementations without breaking code

---

## 📂 File-by-File Breakdown

### Backend Files

#### 1. `SmartSlideGenerator.Api/Program.cs`

**Purpose**: Application startup and configuration

**What It Does**:
```csharp
var builder = WebApplication.CreateBuilder(args);
```
- Creates the application builder
- Reads configuration from appsettings.json

```csharp
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
```
- Registers MVC controllers (endpoints)
- Adds Swagger documentation (automatic API docs)

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
    {
        policy.WithOrigins("http://localhost:5173", "http://localhost:3000")
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});
```
- **CORS Setup**: Allows frontend to talk to backend
- Without this: Browser blocks requests (same-origin policy)
- `5173`: SvelteKit default port
- `3000`: Alternative frontend port

```csharp
builder.Services.AddScoped<ILLMService, KimiLLMService>();
builder.Services.AddScoped<IPowerPointService, PowerPointService>();
```
- **Dependency Injection**: Registers services
- `ILLMService` interface → `KimiLLMService` implementation
- `Scoped`: One instance per HTTP request
- When a controller asks for `ILLMService`, .NET automatically provides `KimiLLMService`

**Why This Pattern?**
- Easy to swap implementations (test with mock, use real in production)
- Decouples code (controllers don't know implementation details)
- Testable (can inject fake services)

---

#### 2. `SmartSlideGenerator.Api/appsettings.json`

**Purpose**: Application configuration

**What It Contains**:
```json
{
  "Logging": { /* Logging settings */ },
  "AllowedHosts": "*",
  "Kimi": {
    "ApiKey": "your-kimi-api-key-here",
    "Model": "moonshot-v1",
    "BaseUrl": "https://api.moonshot.cn/v1"
  },
  "Storage": {
    "BasePath": "./uploads"
  }
}
```

**How It's Used**:
```csharp
var kimiConfig = configuration.GetSection("Kimi");
_apiKey = kimiConfig["ApiKey"];  // Gets from JSON
```

**Security Note**: Never commit real API keys! Use:
- Environment variables in production
- Secrets manager (Azure Key Vault, etc.)
- .gitignore to exclude this file

---

#### 3. `SmartSlideGenerator.Api/Controllers/HealthController.cs`

**Purpose**: Test if API is running

**Code Breakdown**:
```csharp
[ApiController]
[Route("api/[controller]")]
public class HealthController : ControllerBase
```
- `[ApiController]`: Tells .NET this is an API controller
- `[Route("api/[controller]")]`: Base URL is `/api/health`
  - `[controller]` → "Health"
  - Full route: `GET /api/health`

```csharp
[HttpGet]
public IActionResult Get()
{
    return Ok(new { status = "API is running!", timestamp = DateTime.UtcNow });
}
```
- `[HttpGet]`: Responds to GET requests
- Returns JSON: `{ "status": "API is running!", "timestamp": "..." }`
- `Ok()` → HTTP 200 response

**Use This To**:
- Check if API is online
- Verify CORS is working
- Test Swagger UI

---

#### 4. `SmartSlideGenerator.Api/Controllers/PresentationController.cs`

**Purpose**: Main endpoint for presentation generation

**Code Breakdown**:
```csharp
[HttpPost("generate")]
public async Task<IActionResult> GeneratePresentation([FromBody] PresentationRequest request)
```
- `[HttpPost("generate")]`: POST to `/api/presentation/generate`
- `[FromBody]`: Expects JSON in request body
- `async`: Non-blocking operation (waits for AI without freezing)

```csharp
var slides = await _llmService.GenerateSlideContent(
    request.Topic, 
    request.SlideCount, 
    request.Language
);
```
- Calls KimiLLMService to generate slides
- `await`: Waits for AI response
- Returns list of `SlideModel` objects

```csharp
var presentation = new PresentationModel
{
    Title = request.Topic,
    Description = request.Description,
    SlideCount = slides.Count,
    Slides = slides
};
```
- Creates presentation object
- Populates with generated slides

```csharp
var layoutConfig = _layoutService.GetLayoutConfiguration(request.LayoutTemplate);
```
- Gets layout rules (fonts, margins, colors)
- `request.LayoutTemplate` can be "default", "professional", "creative"

---

#### 5. `SmartSlideGenerator.Core/Models/PresentationRequest.cs`

**Purpose**: Define input data structure

```csharp
public class PresentationRequest
{
    public required string Topic { get; set; }
    public string? Description { get; set; }
    public int SlideCount { get; set; } = 5;
    public string LayoutTemplate { get; set; } = "default";
    public string? Language { get; set; } = "en";
}
```

**Explanation**:
- `required`: Must be provided (Topic)
- `?`: Optional (Description, Language)
- `= value`: Default value (SlideCount = 5)

**Example Usage**:
```json
{
  "topic": "Machine Learning",
  "slideCount": 5,
  "layoutTemplate": "professional"
}
```

---

#### 6. `SmartSlideGenerator.Core/Models/PresentationModel.cs`

**Purpose**: Data models for presentation and slides

```csharp
public class PresentationModel
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public required string Title { get; set; }
    public int SlideCount { get; set; }
    public List<SlideModel> Slides { get; set; } = new();
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

**Explanation**:
- `Guid Id`: Unique identifier (like UUID)
- `List<SlideModel>`: Collection of slides
- `DateTime CreatedAt`: When created (auto-filled with current time)

```csharp
public class SlideModel
{
    public int Order { get; set; }
    public string? Title { get; set; }
    public string? Content { get; set; }
    public List<string> BulletPoints { get; set; } = new();
    public string LayoutType { get; set; } = "title-content";
}
```

**Explanation**:
- `Order`: Slide number (1, 2, 3...)
- `BulletPoints`: List of bullet points
- `LayoutType`: Visual layout ("title", "bullets", "two-column")

---

#### 7. `SmartSlideGenerator.Core/Interfaces/ILLMService.cs`

**Purpose**: Define AI service contract

```csharp
public interface ILLMService
{
    Task<List<SlideModel>> GenerateSlideContent(string topic, int slideCount, string? language = "en");
    Task<string> EnhanceContent(string content);
}
```

**Why Interfaces?**
- Decouples code from implementation
- Easy to swap Kimi for GPT, Claude, etc.
- Easy to test with fake AI service
- Multiple classes can implement same interface

**In Dependency Injection**:
```csharp
builder.Services.AddScoped<ILLMService, KimiLLMService>();
```
- Controllers ask for `ILLMService`
- .NET provides `KimiLLMService` instance
- Tomorrow: Can change to `ClaudeService` without breaking controllers

---

#### 8. `SmartSlideGenerator.Infrastructure/LLM/KimiLLMService.cs`

**Purpose**: Actual implementation of AI integration

**Constructor**:
```csharp
public KimiLLMService(IConfiguration configuration, ILogger<KimiLLMService> logger)
{
    _httpClient = new HttpClient();
    var kimiConfig = configuration.GetSection("Kimi");
    _apiKey = kimiConfig["ApiKey"];
    _baseUrl = kimiConfig["BaseUrl"];
}
```
- Gets API key from configuration
- Creates HTTP client for API calls
- Sets up logger for debugging

**Main Method**:
```csharp
public async Task<List<SlideModel>> GenerateSlideContent(string topic, int slideCount, string? language = "en")
```

**Step-by-Step Flow**:

1. **Create Prompt**:
```csharp
var prompt = GeneratePrompt(topic, slideCount, language);
```
- Crafts instruction for AI
- Tells it to return specific JSON format

2. **Build Request**:
```csharp
var requestBody = new
{
    model = _model,
    messages = new[] {
        new { role = "user", content = prompt }
    },
    temperature = 0.7,
    max_tokens = 4000
};
```
- `model`: Which AI model to use
- `messages`: Chat history (single user message)
- `temperature`: 0.7 = balanced creativity vs accuracy
- `max_tokens`: Max response length

3. **Make HTTP Request**:
```csharp
var request = new HttpRequestMessage(HttpMethod.Post, $"{_baseUrl}/chat/completions");
request.Headers.Add("Authorization", $"Bearer {_apiKey}");
request.Content = JsonContent.Create(requestBody);
var response = await _httpClient.SendAsync(request);
```
- POST to Kimi API
- Includes API key in Authorization header
- Sends JSON body
- Waits for response asynchronously

4. **Parse Response**:
```csharp
var slides = await ParseKimiResponse(response.Content);
```
- Extracts JSON from Kimi's response
- Converts to `SlideModel` objects

---

#### 9. `SmartSlideGenerator.Core/Services/LayoutService.cs`

**Purpose**: Manage slide templates and styling

**Constructor**:
```csharp
public LayoutService()
{
    InitializeTemplates();
}

private void InitializeTemplates()
{
    _templates["default"] = new
    {
        Name = "Default",
        Colors = new { Primary = "#2563EB", Secondary = "#1E40AF" },
        FontFamily = "Segoe UI",
        TitleSize = 44,
        BodySize = 18
    };
    // ... more templates
}
```

**Method**:
```csharp
public dynamic GetLayoutConfiguration(string templateName)
{
    return _templates.TryGetValue(templateName.ToLower(), out var template)
        ? template
        : _templates["default"];
}
```
- Looks up template by name
- Returns it if found
- Falls back to "default" if not found
- `TryGetValue`: Safe dictionary access

---

#### 10. `SmartSlideGenerator.Infrastructure/Storage/StorageService.cs`

**Purpose**: Handle file storage

**Constructor**:
```csharp
public StorageService(IConfiguration configuration, ILogger<StorageService> logger)
{
    _basePath = configuration["Storage:BasePath"] ?? "./uploads";
    Directory.CreateDirectory(_basePath);  // Create folder if missing
}
```

**Methods**:
```csharp
public async Task<string> SaveFile(byte[] fileContent, string fileName)
{
    var filePath = Path.Combine(_basePath, fileName);
    await File.WriteAllBytesAsync(filePath, fileContent);
    return filePath;
}
```
- `byte[]`: Raw file data
- `Path.Combine`: Safely join paths
- `async`: Non-blocking file write

```csharp
public async Task<byte[]> GetFile(string filePath)
{
    if (!File.Exists(filePath))
        throw new FileNotFoundException($"File not found: {filePath}");
    
    return await File.ReadAllBytesAsync(filePath);
}
```
- Checks file exists before reading
- Throws exception if not found
- Reads file asynchronously

---

#### 11. `SmartSlideGenerator.Layout/LayoutEngine.cs`

**Purpose**: Orchestrate layout decisions

```csharp
public LayoutConfiguration CreateLayout(string templateName, object content)
{
    var layoutType = _layoutSelector.SelectLayout(content);  // Which layout?
    var rules = _layoutRules.GetRules(layoutType);           // Get rules
    var positions = _elementPositioner.CalculatePositions(content, rules);  // Where?
    
    return new LayoutConfiguration
    {
        LayoutType = layoutType,
        Rules = rules,
        ElementPositions = positions
    };
}
```

**Workflow**:
1. Analyze content → Choose layout type
2. Get rules for that layout
3. Calculate element positions
4. Return complete layout config

---

#### 12. `SmartSlideGenerator.Layout/LayoutRules.cs`

**Purpose**: Define constraints for each layout type

```csharp
_rules["title"] = new LayoutRule
{
    Type = "title",
    MaxLines = 2,
    TitleFontSize = 44,
    SubtitleFontSize = 28,
    Margins = new { Top = 100, Bottom = 100, Left = 50, Right = 50 }
};
```

**Rules for Each Type**:
- **title**: Title slide (big fonts, centered)
- **title-content**: Title + bullet points (most common)
- **bullets**: Just bullet points
- **two-column**: Left/right layout

**Used By**: PowerPoint service to format slides

---

#### 13. `.gitignore`

**Purpose**: Tell Git what NOT to commit

```
bin/          # Compiled C# files
obj/          # Build objects
.vs/          # Visual Studio
node_modules/ # npm packages
.env          # Environment variables (secrets!)
uploads/      # Generated files
```

**Why?**
- Never commit API keys
- Never commit node_modules (huge!)
- Never commit build artifacts

---

### Frontend Files

#### 1. `SmartSlideGenerator-frontend/package.json`

**Purpose**: Node.js project configuration

```json
{
  "name": "smart-slide-generator-frontend",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "svelte": "^4.2.2"
  },
  "devDependencies": {
    "@sveltejs/vite-plugin-svelte": "^3.0.0",
    "vite": "^5.0.0"
  }
}
```

**Explanation**:
- `scripts`: Commands you can run
  - `npm run dev`: Start development server
  - `npm run build`: Create production build
- `dependencies`: Required for runtime
- `devDependencies`: Only needed during development

---

#### 2. `SmartSlideGenerator-frontend/index.html`

**Purpose**: HTML entry point

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Smart Slide Generator</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

**Explanation**:
- `<div id="app"></div>`: Where Vue mounts
- `<script src="/src/main.js"></script>`: Loads JavaScript

---

#### 3. `SmartSlideGenerator-frontend/src/main.js`

**Purpose**: JavaScript entry point

```javascript
import App from './App.svelte'

const app = new App({
  target: document.getElementById('app'),
})

export default app
```

**What It Does**:
1. Imports root component (`App.svelte`)
2. Creates instance
3. Mounts to `#app` div
4. Svelte takes over rendering

---

#### 4. `SmartSlideGenerator-frontend/src/App.svelte`

**Purpose**: Root component

```svelte
<script>
  import Home from './routes/+page.svelte';
</script>

<Home />

<style>
  :global(*) {
    box-sizing: border-box;
  }
</style>
```

**Explanation**:
- Imports Home component
- Renders it
- `:global(*)`: Applies styles globally

---

#### 5. `SmartSlideGenerator-frontend/src/routes/+page.svelte`

**Purpose**: Home page component

**Structure**:
```svelte
<script>
  // Logic
  let topic = '';
  let loading = false;
  
  async function handleGeneratePresentation() {
    // Send to API
    const response = await fetch('http://localhost:5000/api/presentation/generate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ topic, slideCount })
    });
  }
</script>

<div class="container">
  <!-- HTML -->
  <h1>Smart Slide Generator</h1>
  <form on:submit|preventDefault={handleGeneratePresentation}>
    <input bind:value={topic} />
    <button disabled={loading}>
      {loading ? 'Generating...' : 'Generate'}
    </button>
  </form>
</div>

<style>
  /* Styling */
  .container { max-width: 600px; }
</style>
```

**Key Features**:
- `bind:value={topic}`: Two-way data binding
- `on:submit|preventDefault`: Form submission
- `{loading ? 'Generating...' : 'Generate'}`: Ternary in template
- `disabled={loading}`: Disable button while loading
- Beautiful gradient background and styling

---

#### 6. `SmartSlideGenerator-frontend/vite.config.js`

**Purpose**: Vite configuration

```javascript
import { defineConfig } from 'vite'
import { svelte } from '@sveltejs/vite-plugin-svelte'

export default defineConfig({
  plugins: [svelte()],
  server: {
    port: 5173,
    open: true
  }
})
```

**Explanation**:
- `plugins`: Uses Svelte plugin
- `server.port`: Runs on 5173
- `server.open`: Auto-opens browser

---

## 🚀 Step-by-Step Setup

### Prerequisites

```bash
# Check .NET installed
dotnet --version
# Should output: 8.0.x or higher

# Check Node installed
node --version npm --version
# Should output: v18+ and 9+
```

### Backend Setup

```bash
# 1. Navigate to API
cd SmartSlideGenerator.Api

# 2. Restore packages
dotnet restore
# What it does:
# - Reads SmartSlideGenerator.Api.csproj
# - Downloads all NuGet packages (like npm install)
# - Creates ./obj directory

# 3. Build
dotnet build
# What it does:
# - Compiles C# to IL (Intermediate Language)
# - Checks for errors
# - Creates ./bin/Debug directory

# 4. Run
dotnet run
# What it does:
# - Runs Program.cs
# - Starts HTTP server on http://localhost:5000
# - Watches for file changes
```

**Expected Output**:
```
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: https://localhost:5001
      Now listening on: http://localhost:5000
```

### Frontend Setup (New Terminal)

```bash
# 1. Navigate to frontend
cd SmartSlideGenerator-frontend

# 2. Install dependencies
npm install
# What it does:
# - Reads package.json
# - Downloads npm packages to node_modules
# - Creates package-lock.json

# 3. Start dev server
npm run dev
# What it does:
# - Runs Vite dev server
# - Hot reloads on file changes
# - Opens browser at http://localhost:5173
```

**Expected Output**:
```
VITE v5.0.0 ready in 234 ms
Local:    http://localhost:5173/
```

### Configure Kimi API

**Step 1: Get API Key**
1. Go to https://platform.moonshot.cn/
2. Click "Sign Up"
3. Create account (email/phone)
4. Verify email
5. Go to API Keys section
6. Create new key
7. Copy it (save somewhere safe!)

**Step 2: Add to appsettings.json**
```json
{
  "Kimi": {
    "ApiKey": "sk-xxxxxxxxxxxxxxxxxxxx",
    "Model": "moonshot-v1",
    "BaseUrl": "https://api.moonshot.cn/v1"
  }
}
```

---

## 🔄 How Everything Works

### Request Flow (Step-by-Step)

```
1. USER ENTERS TOPIC
   ↓
   Frontend: "Machine Learning"
   ↓

2. USER CLICKS BUTTON
   ↓
   Svelte calls handleGeneratePresentation()
   ↓

3. JAVASCRIPT SENDS HTTP REQUEST
   ↓
   POST http://localhost:5000/api/presentation/generate
   {
     "topic": "Machine Learning",
     "slideCount": 5,
     "layoutTemplate": "default"
   }
   ↓

4. BACKEND RECEIVES REQUEST
   ↓
   PresentationController.GeneratePresentation()
   ↓

5. DI PROVIDES SERVICES
   ↓
   ILLMService → KimiLLMService
   ILayoutService → LayoutService
   IPowerPointService → PowerPointService
   ↓

6. CALL KIMI AI
   ↓
   KimiLLMService.GenerateSlideContent()
   └─→ Creates prompt
   └─→ Sends to Kimi API
   └─→ Waits for response
   └─→ Parses JSON
   └─→ Returns SlideModel list
   ↓

7. APPLY LAYOUT
   ↓
   LayoutService.GetLayoutConfiguration("default")
   └─→ Gets colors, fonts, margins
   ↓

8. CREATE PRESENTATION
   ↓
   PowerPointService.CreatePresentation()
   └─→ Combines slides + layout
   └─→ Generates .pptx file (future)
   ↓

9. SAVE FILE
   ↓
   StorageService.SaveFile()
   └─→ Writes to ./uploads folder
   ↓

10. RETURN RESPONSE
    ↓
    HTTP 200 OK
    {
      "success": true,
      "slideCount": 5,
      "filePath": "./uploads/Machine_Learning_xxxxx.pptx"
    }
    ↓

11. FRONTEND RECEIVES
    ↓
    JavaScript receives response
    ↓

12. SHOW SUCCESS
    ↓
    Svelte displays: "✅ Presentation generated with 5 slides!"
    ↓

13. USER DOWNLOADS
    ↓
    (Future feature)
```

---

## 🐛 Common Errors & Solutions

### Error 1: "API not running"

**Symptom**: Frontend shows "Failed to connect"

**Causes**:
- API didn't start
- Running on wrong port
- CORS not configured

**Solutions**:
```bash
# 1. Check API is running
cd SmartSlideGenerator.Api
dotnet run

# 2. Verify port
# Should see: "Now listening on: http://localhost:5000"

# 3. Check CORS in Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
    {
        policy.WithOrigins("http://localhost:5173")
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

# 4. If port 5000 taken
dotnet run --urls "http://localhost:5001"
```

---

### Error 2: "Kimi API error"

**Symptom**: "Kimi API error: 401"

**Causes**:
- Wrong API key
- API key not set
- Rate limit exceeded

**Solutions**:
```bash
# 1. Check API key
# Open: SmartSlideGenerator.Api/appsettings.json
# Verify: "ApiKey": "sk-xxxx..." is correct

# 2. Test key manually
curl -H "Authorization: Bearer sk-xxxx" \
  https://api.moonshot.cn/v1/chat/completions

# 3. Check free tier limits
# Kimi free tier: Limited requests/day
# Wait 24 hours or upgrade

# 4. Check logs
# Watch console output for error details
```

---

### Error 3: "Port already in use"

**Symptom**: "Address already in use"

**Solution**:
```bash
# Option 1: Kill process on port 5000
# Windows
netstat -ano | findstr :5000
taskkill /PID <PID> /F

# macOS/Linux
lsof -i :5000
kill -9 <PID>

# Option 2: Use different port
dotnet run --urls "http://localhost:5001"
```

---

### Error 4: "npm install fails"

**Symptom**: "ERR! 404 not found"

**Solution**:
```bash
# 1. Clear npm cache
npm cache clean --force

# 2. Delete node_modules
rm -rf node_modules package-lock.json

# 3. Reinstall
npm install

# 4. Check Node version
node --version  # Should be 18+
npm --version   # Should be 9+
```

---

### Error 5: "CORS error"

**Symptom**: "Access to XMLHttpRequest blocked by CORS policy"

**Cause**: Frontend URL not in CORS whitelist

**Solution**:
```csharp
// In Program.cs, update this:
policy.WithOrigins("http://localhost:5173")  // Your frontend URL
      .AllowAnyMethod()
      .AllowAnyHeader();

// Restart API after changes!
```

---

## 🧪 Testing Guide

### Test 1: API Health Check

```bash
curl http://localhost:5000/api/health
```

**Expected Response**:
```json
{
  "status": "API is running!",
  "timestamp": "2026-07-10T10:51:00Z"
}
```

---

### Test 2: Generate Presentation

```bash
curl -X POST http://localhost:5000/api/presentation/generate \
  -H "Content-Type: application/json" \
  -d '{
    "topic": "Python Programming",
    "slideCount": 3,
    "layoutTemplate": "default"
  }'
```

**Expected Response**:
```json
{
  "success": true,
  "slideCount": 3,
  "filePath": "./uploads/Python_Programming_xxxxx.pptx",
  "message": "Presentation generated successfully"
}
```

---

### Test 3: Swagger UI

1. Open: http://localhost:5000/swagger
2. See all endpoints documented
3. Click "Try it out" to test endpoints
4. View request/response

---

### Test 4: Kimi API Directly

```bash
curl -X POST https://api.moonshot.cn/v1/chat/completions \
  -H "Authorization: Bearer sk-xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "moonshot-v1",
    "messages": [
      {"role": "user", "content": "Hello"}
    ]
  }'
```

---

## 📊 File Summary

| File | Lines | Purpose |
|------|-------|---------|
| Program.cs | 45 | Startup & DI config |
| appsettings.json | 15 | Configuration |
| HealthController.cs | 15 | Health endpoint |
| PresentationController.cs | 60 | Main endpoint |
| Models | 40 | Data structures |
| Services | 150 | Business logic |
| Layout | 100 | Layout system |
| Frontend | 200 | UI components |

---

## 🎓 Next Steps

1. **Run it locally** (follow setup guide)
2. **Test endpoints** (use Swagger or curl)
3. **Understand flow** (trace a request end-to-end)
4. **Implement PPTX** (add DocumentFormat.OpenXml)
5. **Add database** (PostgreSQL + EF Core)
6. **Deploy** (Docker + Cloud)

---

## 📞 Getting Help

### How to Debug

1. **Check logs** - Frontend console and API logs
2. **Use Swagger** - http://localhost:5000/swagger
3. **Test API directly** - Use curl or Postman
4. **Review code** - Look at comments and error messages
5. **Ask for help** - Share logs and error messages

---

**🎉 You have a complete, documented AI PowerPoint generator!**

Good luck! 🚀
