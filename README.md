# SmartSlideGenerator

An AI-powered PowerPoint presentation generator using C# backend, Kimi AI, and SvelteKit frontend.

## 🎯 Features

- Generate presentations from text prompts
- Multiple slide templates
- AI-powered content generation using Kimi
- Export to PowerPoint format

## 🛠️ Tech Stack

- **Backend**: C# .NET 8
- **Frontend**: SvelteKit + Svelte
- **AI Provider**: Kimi API
- **Database**: PostgreSQL (coming soon)

## 📁 Project Structure

```
SmartSlideGenerator/
├── SmartSlideGenerator.Api/          # Main API (ASP.NET Core)
├── SmartSlideGenerator.Core/         # Business logic
├── SmartSlideGenerator.Infrastructure/ # External services
├── SmartSlideGenerator.Layout/       # Layout engine
└── SmartSlideGenerator-frontend/     # Web UI (SvelteKit)
```

## 🚀 Quick Start

### Backend Setup

```bash
cd SmartSlideGenerator.Api
dotnet restore
dotnet run
```

API runs at: `http://localhost:5000`

### Frontend Setup

```bash
cd SmartSlideGenerator-frontend
npm install
npm run dev
```

Frontend runs at: `http://localhost:5173`

## 📝 Configuration

Update `appsettings.json` with:
- Kimi API key
- Database connection (when ready)

## 📚 Learning Guide

This project is built step-by-step to teach:
- C# and .NET basics
- API design
- Frontend-backend communication
- AI integration
- Clean architecture

## 📖 Documentation

See [DEVELOPMENT.md](DEVELOPMENT.md) for detailed setup instructions.

## 📄 License

MIT
