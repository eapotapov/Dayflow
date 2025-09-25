# Dayflow Developer Documentation

## Overview

This directory contains comprehensive technical documentation for developers working on the Dayflow codebase. Dayflow is a macOS application that records screen activity, analyzes it with AI, and presents an intelligent timeline of user activities.

## Documentation Index

### Core Documentation

1. **[Architecture Overview](architecture.md)**
   - System architecture and design patterns
   - Component relationships and responsibilities
   - Key design decisions and rationale
   - Extension points for new features

2. **[Database Schema](database.md)**
   - Complete SQLite schema documentation
   - Table relationships and data lifecycle
   - Query examples and optimization tips
   - Migration and backup procedures

3. **[Data Flow](data-flow.md)**
   - End-to-end data pipeline from capture to display
   - Transformation stages and formats
   - Performance metrics and volumes
   - Error handling and recovery

### AI/LLM Documentation

4. **[Prompting Structure](prompting.md)**
   - LLM prompt engineering guidelines
   - Two-phase analysis approach
   - Provider-specific implementations
   - Quality control and optimization

5. **[API Providers](api-providers.md)**
   - Supported LLM providers (Gemini, Ollama, LM Studio)
   - Configuration and authentication
   - Performance comparison and costs
   - Adding new providers

### Feature Documentation

6. **[Categories System](categories.md)**
   - Default categories and customization
   - AI category assignment logic
   - User interface integration
   - Best practices for category design

7. **[External Connections](external-connections.md)**
   - All network connections and APIs
   - Privacy and security implications
   - Monitoring and debugging network traffic
   - Offline mode configuration

### Getting Started

8. **[Developer Setup Guide](setup-guide.md)**
   - Prerequisites and system requirements
   - Fork and repository setup
   - Xcode configuration and signing
   - API keys and local development
   - Common issues and solutions

## Quick Start

```bash
# Clone the repository
git clone git@github.com:YOUR_USERNAME/Dayflow.git
cd Dayflow

# Open in Xcode
open Dayflow/Dayflow.xcodeproj

# Configure your development team and bundle ID
# Add GEMINI_API_KEY to scheme environment variables
# Build and run (⌘R)
```

## Key Technologies

- **Language**: Swift 5.9+
- **UI Framework**: SwiftUI
- **Database**: GRDB (SQLite)
- **Screen Recording**: ScreenCaptureKit
- **AI Providers**: Gemini API, Ollama, LM Studio
- **Analytics**: PostHog
- **Updates**: Sparkle

## Repository Structure

```
Dayflow/
├── docs/developer/        # This documentation
├── Dayflow/              # Main application code
│   ├── App/              # Application lifecycle
│   ├── Core/             # Business logic
│   ├── Models/           # Data structures
│   ├── Views/            # User interface
│   ├── System/           # Platform integration
│   └── Utilities/        # Helper functions
├── scripts/              # Build and release scripts
└── CLAUDE.md             # AI assistant instructions
```

## Development Workflow

1. **Make changes** in your fork
2. **Test thoroughly** with different providers
3. **Document** significant changes
4. **Sync with upstream** regularly
5. **Create pull requests** for contributions

## Common Tasks

### Running Tests
```bash
# In Xcode
cmd+U

# From command line
xcodebuild test -scheme Dayflow
```

### Viewing Logs
```bash
# Filter Console.app by process
log show --predicate 'process == "Dayflow"' --last 1h

# View database logs
sqlite3 ~/Library/Application\ Support/Dayflow/chunks.sqlite
```

### Debugging LLM Calls
```swift
// Enable debug logging
UserDefaults.standard.set(true, forKey: "llmDebugLogging")

// Check failed calls in database
SELECT * FROM llm_calls WHERE status = 'failure';
```

## Contributing

When contributing to Dayflow:

1. **Follow existing patterns** - Maintain consistency with current code style
2. **Add tests** - Cover new functionality with appropriate tests
3. **Update documentation** - Keep docs in sync with code changes
4. **Consider privacy** - Minimize data collection and external connections
5. **Think about performance** - Profile and optimize intensive operations

## Support

- **Original Repository**: https://github.com/JerryZLiu/Dayflow
- **Issues**: Report bugs and request features
- **Discussions**: Ask questions and share ideas
- **Documentation Updates**: Submit PRs for doc improvements

## License

Dayflow is released under the MIT License. See the LICENSE file in the root directory for details.

---

*Last updated: January 2025*
*Documentation version: 1.0*