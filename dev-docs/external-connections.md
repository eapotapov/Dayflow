# External Connections Documentation

## Overview

This document details all network connections made by Dayflow, including APIs, services, and external resources. Understanding these connections is crucial for security, privacy, and debugging.

## Connection Summary

| Service | Purpose | Frequency | Data Sent | Can Disable |
|---------|---------|-----------|-----------|-------------|
| Gemini API | Video analysis | Every 15 min | Screen recordings | Yes (use local) |
| PostHog | Analytics | Real-time | Usage events | Yes |
| Sparkle | Auto-updates | Daily | Version check | Yes |
| Google S2 | Favicons | On demand | Domain names | No |
| Direct sites | Favicons | On demand | Domain names | No |

## Detailed Connections

### 1. Google Gemini API

**Purpose**: Analyze screen recordings and generate activity summaries

**Endpoints**:
```
POST https://generativelanguage.googleapis.com/upload/v1beta/files
POST https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-pro:generateContent
POST https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent
```

**Authentication**: API key in query parameter
```
?key={GEMINI_API_KEY}
```

**Data Transmitted**:
- **Upload**: Complete video file (15 minutes, ~5MB)
- **Analysis**: Video URI + prompt text
- **Response**: JSON with observations and activity cards

**Frequency**:
- Every 15 minutes when recording is active
- ~96 API calls per day

**Privacy Implications**:
- Screen content sent to Google servers
- Processed according to Gemini API terms
- Can be retained for model improvement (unless paid tier)

**Disable Option**:
```swift
// Switch to local provider in Settings
UserDefaults.standard.set("ollamaLocal", forKey: "selectedLLMProvider")
```

### 2. PostHog Analytics

**Purpose**: Product analytics and usage tracking

**Endpoint**:
```
POST https://us.i.posthog.com/capture/
POST https://us.i.posthog.com/decide/
```

**Authentication**: API key in request body
```json
{
  "api_key": "phc_...",
  "distinct_id": "uuid-from-keychain"
}
```

**Data Transmitted**:
```json
{
  "event": "analysis_batch_completed",
  "properties": {
    "batch_id": 123,
    "cards_generated": 3,
    "processing_duration_seconds": 8,
    "os_version": "14.0",
    "app_version": "1.0.31"
  }
}
```

**Events Tracked**:
- App launch/quit
- Recording start/stop
- Analysis success/failure
- Screen views
- Feature usage

**Frequency**: Real-time as events occur

**Privacy Implications**:
- Anonymous UUID (not linked to identity)
- No PII transmitted
- System info and usage patterns collected

**Disable Option**:
```swift
// In Settings or programmatically
AnalyticsService.shared.setOptIn(false)
```

### 3. Sparkle Auto-Updates

**Purpose**: Check for and download app updates

**Endpoint**:
```
GET https://dayflow.so/appcast.xml
GET https://github.com/JerryZLiu/Dayflow/releases/download/...
```

**Data Transmitted**:
- Current app version
- System version
- Locale
- CPU architecture

**Response**: XML appcast with available updates
```xml
<item>
  <title>Version 1.0.32</title>
  <sparkle:version>132</sparkle:version>
  <link>https://github.com/.../Dayflow.dmg</link>
  <sparkle:edSignature>...</sparkle:edSignature>
</item>
```

**Frequency**:
- Automatic check daily
- Manual check on demand
- Background download when update available

**Privacy Implications**:
- Version information shared
- Download analytics to GitHub
- No user data transmitted

**Disable Option**:
```swift
// Disable automatic checks
SUUpdater.shared.automaticallyChecksForUpdates = false
```

### 4. Favicon Services

#### Google S2 Favicons
**Endpoint**:
```
GET https://www.google.com/s2/favicons?domain={domain}&sz=64
```

**Purpose**: Fetch website icons for timeline display

**Data Transmitted**:
- Domain name only (no paths or queries)
- Icon size preference

**Privacy Implications**:
- Google knows which domains appear in your timeline
- No authentication required
- Cached locally after first fetch

#### Direct Site Favicons
**Endpoint**:
```
GET https://{domain}/favicon.ico
```

**Purpose**: Fallback favicon fetching

**Data Transmitted**:
- HTTP request to site
- User agent string

**Privacy Implications**:
- Sites know Dayflow requested their favicon
- Could potentially track usage
- No authentication

### 5. Local AI Providers

#### Ollama
**Endpoint**:
```
POST http://localhost:11434/api/generate
POST http://localhost:11434/api/tags
```

**Data Transmitted**:
- Video frames as base64
- Prompt text
- Model parameters

**Privacy**: All processing local, no external transmission

#### LM Studio
**Endpoint**:
```
POST http://localhost:1234/v1/completions
```

**Data Transmitted**: Same as Ollama

**Privacy**: All processing local, no external transmission

### 6. External Links (User-Initiated)

**Purpose**: Open documentation and tools in browser

**URLs**:
```
https://github.com/JerryZLiu/Dayflow
https://aistudio.google.com/app/apikey
https://ollama.com/download
https://lmstudio.ai/
```

**Data Transmitted**: None (opens in default browser)

## Network Security

### HTTPS Enforcement
All external connections use HTTPS except:
- Local AI providers (localhost only)

### Certificate Validation
```swift
// Default URLSession configuration
let config = URLSessionConfiguration.default
// Certificate pinning not implemented
```

### API Key Security
```swift
// Stored in Keychain
KeychainManager.shared.store(apiKey, for: "gemini")

// Never in:
- UserDefaults
- Plist files
- Source code
- Log files
```

## Data Minimization

### What's Never Sent
- User identity
- Email addresses
- File paths
- Personal information
- Raw database content

### What's Anonymized
- Analytics events (UUID only)
- Error reports (sanitized)
- Performance metrics (aggregated)

## Monitoring Connections

### Network Debugging
```bash
# Monitor all connections
nettop -P Dayflow

# Check specific domains
tcpdump -i any host generativelanguage.googleapis.com

# Proxy through Charles/Proxyman
export HTTPS_PROXY=http://localhost:8888
```

### In-App Logging
```swift
// Enable network logging
UserDefaults.standard.set(true, forKey: "networkDebugLogging")

// View in Console.app
log.debug("Network request to: \(url)")
```

### Database Logging
```sql
-- View LLM API calls
SELECT * FROM llm_calls ORDER BY created_at DESC;

-- Check request/response
SELECT request_url, http_status, latency_ms FROM llm_calls;
```

## Firewall Rules

### Required for Core Functionality
```
# Gemini API
ALLOW generativelanguage.googleapis.com:443

# Local AI
ALLOW localhost:11434
ALLOW localhost:1234
```

### Optional Services
```
# Analytics
ALLOW us.i.posthog.com:443

# Updates
ALLOW dayflow.so:443
ALLOW github.com:443

# Favicons
ALLOW www.google.com:443
ALLOW *:443 (for direct favicon fetch)
```

### Fully Offline Mode
Block all except:
```
ALLOW localhost:*
```

## Privacy Settings

### Maximum Privacy Configuration
```swift
// Use local AI
UserDefaults.standard.set("ollamaLocal", forKey: "selectedLLMProvider")

// Disable analytics
AnalyticsService.shared.setOptIn(false)

// Disable auto-updates
SUUpdater.shared.automaticallyChecksForUpdates = false

// Result: Only favicon fetches remain
```

### Data Retention

#### Gemini
- Depends on API tier (free vs paid)
- See Gemini API terms for details

#### PostHog
- 90 days default retention
- Configurable in PostHog settings

#### Local Cache
- Favicons: Cached indefinitely
- API responses: Not cached
- Videos: 3-day retention

## Compliance

### GDPR Considerations
- Right to deletion: Delete Keychain entries and local data
- Right to access: Export database and files
- Consent: Analytics opt-in available

### CCPA Considerations
- No sale of personal information
- Opt-out mechanisms provided
- Transparent about data collection

## Testing Without Network

### Fully Offline Testing
```bash
# Disable network
networksetup -setairportpower Wi-Fi off

# Use local AI provider
# Analytics will fail gracefully
# Updates won't check
```

### Mock Responses
```swift
// For testing, intercept URLSession
class MockURLProtocol: URLProtocol {
    override class func canInit(with request: URLRequest) -> Bool {
        return request.url?.host == "generativelanguage.googleapis.com"
    }

    override func startLoading() {
        // Return mock response
    }
}
```

## Future Considerations

### Planned Connections
- Cloud sync service (optional)
- Backup service (optional)
- Plugin marketplace

### Security Enhancements
- Certificate pinning for APIs
- End-to-end encryption for sync
- Zero-knowledge architecture

## Emergency Procedures

### API Key Compromise
1. Revoke key immediately in provider console
2. Generate new key
3. Update in Keychain
4. No app update required

### Privacy Breach
1. Disable recording
2. Clear all local data
3. Revoke API access
4. Contact support

### Rate Limiting
1. Switch to local provider temporarily
2. Implement request queuing
3. Add exponential backoff