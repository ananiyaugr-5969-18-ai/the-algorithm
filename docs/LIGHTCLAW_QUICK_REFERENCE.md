# LightClaw Quick Reference Guide

## One-Page Overview

LightClaw is a **lightweight alternative to OpenClaw** with native support for Google Antigravity, full MCP implementation, extensible skills, and seamless Slack integration.

---

## Installation

```bash
# Install via pip
pip install lightclaw

# Or from source
git clone https://github.com/your-org/lightclaw.git
cd lightclaw
pip install -e .
```

---

## Quick Start

```bash
# Initialize configuration
lightclaw init

# Edit lightclaw.yaml with your credentials

# Start interactive chat
lightclaw chat "Hello!"

# Run as server
lightclaw serve --port 8080
```

---

## Configuration Template

```yaml
# lightclaw.yaml
version: "1.0"

core:
  name: "My Agent"
  log_level: "info"
  
providers:
  google_antigravity:
    enabled: true
    credentials_file: "./credentials.json"
    project_id: "your-project-id"
    model: "antigravity-v2"
    temperature: 0.7
    max_tokens: 4096

mcp:
  enabled: true
  context_ttl: 3600
  max_context_size: 10000
  
skills:
  enabled: true
  auto_discover: true
  skill_paths: ["./skills"]
  allowed_skills:
    - "web_search"
    - "code_analysis"
    - "slack_messaging"
    
integrations:
  slack:
    enabled: true
    bot_token: "${SLACK_BOT_TOKEN}"
    app_token: "${SLACK_APP_TOKEN}"
    signing_secret: "${SLACK_SIGNING_SECRET}"
```

---

## Python API

### Basic Usage

```python
import asyncio
from lightclaw import LightClawAgent, Config

async def main():
    # Load config
    config = Config.from_file("lightclaw.yaml")
    
    # Create agent
    agent = LightClawAgent(config)
    await agent.initialize()
    
    try:
        # Process request
        response = await agent.process("Hello, world!")
        print(response.output)
        
        # Continue conversation
        response2 = await agent.process(
            "Tell me more",
            context_id=response.context_id
        )
        print(response2.output)
    finally:
        await agent.close()

asyncio.run(main())
```

### With Skills

```python
# Enable specific skills
response = await agent.process(
    prompt="Search for Python trends and post to Slack",
    skills=["web_search", "slack_messaging"]
)
```

### Streaming Responses

```python
async for chunk in agent.stream_process("Explain quantum computing"):
    print(chunk, end="", flush=True)
```

---

## Creating Custom Skills

```python
# skills/my_skill.py
from lightclaw.skills import Skill, SkillResult

class MySkill(Skill):
    def __init__(self):
        super().__init__(
            name="my_skill",
            description="Does something useful",
            version="1.0.0"
        )
    
    async def execute(self, context: dict, params: dict) -> SkillResult:
        try:
            # Your logic here
            result = do_something(params)
            
            return SkillResult(
                success=True,
                output=result
            )
        except Exception as e:
            return SkillResult(
                success=False,
                error=str(e)
            )

def register():
    return MySkill()
```

---

## Slack Integration

### Setup Environment

```bash
export SLACK_BOT_TOKEN="xoxb-..."
export SLACK_APP_TOKEN="xapp-..."
export SLACK_SIGNING_SECRET="..."
```

### Command Handler

```python
from lightclaw.integrations.slack import SlackHandler

@SlackHandler.command("/lightclaw")
async def handle_command(command: dict, agent):
    text = command["text"]
    response = await agent.process(text)
    
    return {
        "response_type": "in_channel",
        "text": response.output
    }
```

### Event Handler

```python
@SlackHandler.event("app_mention")
async def handle_mention(event: dict, agent):
    response = await agent.process(event["text"])
    await agent.slack.post_message(
        channel=event["channel"],
        text=response.output,
        thread_ts=event.get("ts")
    )
```

---

## REST API

### Start Server

```bash
lightclaw serve --port 8080 --host 0.0.0.0
```

### Endpoints

**Process Request**
```bash
curl -X POST http://localhost:8080/api/v1/process \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Hello, world!",
    "skills": ["web_search"],
    "context_id": "optional-context-id"
  }'
```

**List Skills**
```bash
curl http://localhost:8080/api/v1/skills
```

**Execute Skill**
```bash
curl -X POST http://localhost:8080/api/v1/skills/web_search \
  -H "Content-Type: application/json" \
  -d '{
    "params": {
      "query": "Python trends 2024"
    }
  }'
```

**Health Check**
```bash
curl http://localhost:8080/api/v1/health
```

---

## Docker Deployment

### Dockerfile

```dockerfile
FROM python:3.9-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8080
CMD ["lightclaw", "serve", "--port", "8080"]
```

### Build & Run

```bash
# Build
docker build -t lightclaw:latest .

# Run
docker run -d \
  -p 8080:8080 \
  -e SLACK_BOT_TOKEN=$SLACK_BOT_TOKEN \
  -v $(pwd)/config:/app/config \
  -v $(pwd)/credentials:/app/credentials \
  lightclaw:latest
```

### Docker Compose

```yaml
version: '3.8'

services:
  lightclaw:
    image: lightclaw:latest
    ports:
      - "8080:8080"
    environment:
      - SLACK_BOT_TOKEN=${SLACK_BOT_TOKEN}
      - SLACK_APP_TOKEN=${SLACK_APP_TOKEN}
    volumes:
      - ./config:/app/config
      - ./credentials:/app/credentials
      - ./skills:/app/skills
    restart: unless-stopped
```

---

## Environment Variables

```bash
# Slack
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
SLACK_SIGNING_SECRET=...

# Google Cloud
GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json
GOOGLE_CLOUD_PROJECT=your-project-id

# Optional
LIGHTCLAW_CONFIG_PATH=/path/to/lightclaw.yaml
LIGHTCLAW_LOG_LEVEL=info
```

---

## Common Commands

```bash
# Initialize new project
lightclaw init

# Validate configuration
lightclaw validate

# Chat interactively
lightclaw chat

# Process single query
lightclaw query "What's the weather?"

# Start server
lightclaw serve --port 8080

# List available skills
lightclaw skills list

# Test a skill
lightclaw skills test web_search --query "Python"

# Show version
lightclaw version

# Show help
lightclaw --help
```

---

## Troubleshooting

### Issue: Authentication Failed

```bash
# Check credentials file exists
ls -la credentials/google-antigravity.json

# Verify permissions
chmod 600 credentials/google-antigravity.json

# Test authentication
lightclaw test-auth
```

### Issue: Skill Not Found

```bash
# List available skills
lightclaw skills list

# Reload skills
lightclaw skills reload

# Check skill path
echo $LIGHTCLAW_SKILL_PATH
```

### Issue: Slack Connection Failed

```bash
# Verify tokens
echo $SLACK_BOT_TOKEN | cut -c1-10

# Test Slack connection
lightclaw test-slack

# Check Slack app permissions
# Visit: https://api.slack.com/apps
```

---

## Performance Tuning

### Memory Optimization

```yaml
core:
  max_memory_mb: 50  # Limit memory usage

mcp:
  context_ttl: 1800  # Shorter TTL = less memory
  max_context_size: 5000  # Fewer messages
```

### Concurrent Requests

```yaml
core:
  max_workers: 4  # Parallel processing
  request_timeout: 30  # Seconds
```

### Provider Settings

```yaml
providers:
  google_antigravity:
    max_tokens: 2048  # Fewer tokens = faster
    temperature: 0.5  # Lower = more deterministic
    timeout: 10  # Faster timeout
```

---

## Monitoring

### Health Check Endpoint

```bash
curl http://localhost:8080/api/v1/health
```

Response:
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "uptime": 3600,
  "memory_mb": 45,
  "active_contexts": 3,
  "skills_loaded": 5
}
```

### Metrics

```python
# Get agent metrics
metrics = await agent.get_metrics()
print(f"Requests: {metrics.total_requests}")
print(f"Errors: {metrics.total_errors}")
print(f"Avg latency: {metrics.avg_latency_ms}ms")
```

---

## Best Practices

1. **Always close agents**
   ```python
   try:
       await agent.process(...)
   finally:
       await agent.close()
   ```

2. **Use context IDs for conversations**
   ```python
   response = await agent.process("Hello", context_id=ctx_id)
   ```

3. **Enable only needed skills**
   ```python
   response = await agent.process(
       prompt="...",
       skills=["web_search"]  # Not all skills
   )
   ```

4. **Set appropriate timeouts**
   ```yaml
   core:
     request_timeout: 30
   ```

5. **Monitor memory usage**
   ```python
   if agent.memory_mb > 100:
       await agent.cleanup()
   ```

---

## Resources

- **Documentation**: https://docs.lightclaw.io
- **GitHub**: https://github.com/your-org/lightclaw
- **Discord**: https://discord.gg/lightclaw
- **Issues**: https://github.com/your-org/lightclaw/issues
- **Examples**: https://github.com/your-org/lightclaw-examples

---

## Comparison: LightClaw vs OpenClaw

| Metric | LightClaw | OpenClaw |
|--------|-----------|----------|
| Size | 50MB | 200MB |
| Startup | <1s | ~5s |
| Google Antigravity | ✅ | ❌ |
| MCP Full Support | ✅ | ⚠️ |
| Custom Skills | ✅ Easy | ⚠️ Hard |
| Slack Native | ✅ | ⚠️ Plugin |

**Choose LightClaw for**: Google Antigravity, lightweight deployment, custom skills

**Choose OpenClaw for**: Web UI, established community, all-in-one solution

---

## License

MIT License - Free for commercial and personal use

---

## Support

- 📧 Email: support@lightclaw.io
- 💬 Discord: discord.gg/lightclaw
- 🐛 Issues: github.com/your-org/lightclaw/issues
- 📖 Docs: docs.lightclaw.io
