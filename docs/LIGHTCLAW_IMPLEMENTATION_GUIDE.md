# LightClaw Implementation Guide

## Table of Contents
1. [Core Architecture](#core-architecture)
2. [Provider Implementation](#provider-implementation)
3. [MCP Integration](#mcp-integration)
4. [Skills System](#skills-system)
5. [Slack Integration](#slack-integration)
6. [Complete Example](#complete-example)

## Core Architecture

### Project Structure

```
lightclaw/
├── lightclaw/
│   ├── __init__.py
│   ├── core/
│   │   ├── __init__.py
│   │   ├── agent.py           # Main agent class
│   │   ├── config.py          # Configuration management
│   │   └── context.py         # Context management
│   ├── providers/
│   │   ├── __init__.py
│   │   ├── base.py            # Base provider interface
│   │   ├── google_antigravity.py
│   │   └── provider_manager.py
│   ├── mcp/
│   │   ├── __init__.py
│   │   ├── protocol.py        # MCP protocol implementation
│   │   └── context_manager.py
│   ├── skills/
│   │   ├── __init__.py
│   │   ├── base.py            # Base skill interface
│   │   ├── engine.py          # Skills engine
│   │   └── builtin/           # Built-in skills
│   └── integrations/
│       ├── __init__.py
│       └── slack/
│           ├── __init__.py
│           ├── client.py
│           └── handlers.py
├── skills/                     # User-defined skills
├── config/
│   └── lightclaw.yaml
├── requirements.txt
├── setup.py
└── README.md
```

## Provider Implementation

### Base Provider Interface

```python
# lightclaw/providers/base.py

from abc import ABC, abstractmethod
from typing import Dict, Any, Optional, AsyncIterator
from dataclasses import dataclass

@dataclass
class ModelResponse:
    """Response from a model provider"""
    content: str
    tokens_used: int
    finish_reason: str
    metadata: Optional[Dict[str, Any]] = None

class BaseProvider(ABC):
    """Base class for all model providers"""
    
    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.name = self.__class__.__name__
        
    @abstractmethod
    async def initialize(self):
        """Initialize the provider with credentials and settings"""
        pass
    
    @abstractmethod
    async def generate(
        self, 
        prompt: str, 
        context: Optional[Dict[str, Any]] = None,
        **kwargs
    ) -> ModelResponse:
        """
        Generate a response from the model
        
        Args:
            prompt: The input prompt
            context: Optional context for the generation
            **kwargs: Provider-specific parameters
            
        Returns:
            ModelResponse with generated content
        """
        pass
    
    @abstractmethod
    async def stream_generate(
        self,
        prompt: str,
        context: Optional[Dict[str, Any]] = None,
        **kwargs
    ) -> AsyncIterator[str]:
        """
        Stream generated response in chunks
        
        Args:
            prompt: The input prompt
            context: Optional context for the generation
            **kwargs: Provider-specific parameters
            
        Yields:
            String chunks of generated content
        """
        pass
    
    @abstractmethod
    async def close(self):
        """Clean up provider resources"""
        pass
```

### Google Antigravity Provider Implementation

```python
# lightclaw/providers/google_antigravity.py

import json
from typing import Dict, Any, Optional, AsyncIterator
from google.oauth2 import service_account
from google.auth.transport.requests import Request
import aiohttp

from .base import BaseProvider, ModelResponse

class GoogleAntigravityProvider(BaseProvider):
    """Provider for Google Antigravity AI models"""
    
    API_ENDPOINT = "https://antigravity-api.googleapis.com/v1"
    
    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)
        self.credentials = None
        self.session = None
        self.project_id = config.get("project_id")
        self.model = config.get("model", "antigravity-v2")
        
    async def initialize(self):
        """Initialize Google Antigravity credentials"""
        credentials_file = self.config.get("credentials_file")
        
        if credentials_file:
            # Load service account credentials
            self.credentials = service_account.Credentials.from_service_account_file(
                credentials_file,
                scopes=["https://www.googleapis.com/auth/cloud-platform"]
            )
        else:
            raise ValueError("credentials_file is required for Google Antigravity")
        
        # Create aiohttp session
        self.session = aiohttp.ClientSession()
        
    async def _get_access_token(self) -> str:
        """Get a valid access token"""
        if not self.credentials.valid:
            self.credentials.refresh(Request())
        return self.credentials.token
    
    async def generate(
        self,
        prompt: str,
        context: Optional[Dict[str, Any]] = None,
        **kwargs
    ) -> ModelResponse:
        """Generate response using Google Antigravity"""
        
        access_token = await self._get_access_token()
        
        # Prepare request payload
        payload = {
            "model": self.model,
            "prompt": prompt,
            "temperature": kwargs.get("temperature", self.config.get("temperature", 0.7)),
            "max_tokens": kwargs.get("max_tokens", self.config.get("max_tokens", 4096)),
            "context": context or {}
        }
        
        # Make API request
        url = f"{self.API_ENDPOINT}/projects/{self.project_id}/generate"
        headers = {
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json"
        }
        
        async with self.session.post(url, json=payload, headers=headers) as response:
            if response.status != 200:
                error_text = await response.text()
                raise Exception(f"Google Antigravity API error: {error_text}")
            
            data = await response.json()
            
            return ModelResponse(
                content=data.get("content", ""),
                tokens_used=data.get("tokens_used", 0),
                finish_reason=data.get("finish_reason", "complete"),
                metadata=data.get("metadata", {})
            )
    
    async def stream_generate(
        self,
        prompt: str,
        context: Optional[Dict[str, Any]] = None,
        **kwargs
    ) -> AsyncIterator[str]:
        """Stream generated response"""
        
        access_token = await self._get_access_token()
        
        payload = {
            "model": self.model,
            "prompt": prompt,
            "temperature": kwargs.get("temperature", self.config.get("temperature", 0.7)),
            "max_tokens": kwargs.get("max_tokens", self.config.get("max_tokens", 4096)),
            "context": context or {},
            "stream": True
        }
        
        url = f"{self.API_ENDPOINT}/projects/{self.project_id}/stream"
        headers = {
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json"
        }
        
        async with self.session.post(url, json=payload, headers=headers) as response:
            async for line in response.content:
                if line:
                    try:
                        data = json.loads(line.decode('utf-8'))
                        if "content" in data:
                            yield data["content"]
                    except json.JSONDecodeError:
                        continue
    
    async def close(self):
        """Close the session"""
        if self.session:
            await self.session.close()
```

## MCP Integration

### MCP Protocol Implementation

```python
# lightclaw/mcp/protocol.py

from typing import Dict, Any, List, Optional
from dataclasses import dataclass, field
from datetime import datetime
import uuid

@dataclass
class MCPMessage:
    """Represents a single message in MCP protocol"""
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    role: str = "user"  # user, assistant, system
    content: str = ""
    timestamp: datetime = field(default_factory=datetime.utcnow)
    metadata: Dict[str, Any] = field(default_factory=dict)

@dataclass
class MCPContext:
    """MCP context containing conversation history and metadata"""
    context_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    messages: List[MCPMessage] = field(default_factory=list)
    metadata: Dict[str, Any] = field(default_factory=dict)
    created_at: datetime = field(default_factory=datetime.utcnow)
    updated_at: datetime = field(default_factory=datetime.utcnow)
    
    def add_message(self, role: str, content: str, metadata: Optional[Dict] = None):
        """Add a message to the context"""
        message = MCPMessage(
            role=role,
            content=content,
            metadata=metadata or {}
        )
        self.messages.append(message)
        self.updated_at = datetime.utcnow()
        return message
    
    def get_recent_messages(self, count: int = 10) -> List[MCPMessage]:
        """Get the most recent N messages"""
        return self.messages[-count:]
    
    def to_dict(self) -> Dict[str, Any]:
        """Convert context to dictionary"""
        return {
            "context_id": self.context_id,
            "messages": [
                {
                    "id": msg.id,
                    "role": msg.role,
                    "content": msg.content,
                    "timestamp": msg.timestamp.isoformat(),
                    "metadata": msg.metadata
                }
                for msg in self.messages
            ],
            "metadata": self.metadata,
            "created_at": self.created_at.isoformat(),
            "updated_at": self.updated_at.isoformat()
        }

class MCPProtocol:
    """Implementation of Model Context Protocol"""
    
    VERSION = "1.0"
    
    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.contexts: Dict[str, MCPContext] = {}
        self.context_ttl = config.get("context_ttl", 3600)
        self.max_context_size = config.get("max_context_size", 10000)
    
    def create_context(self, metadata: Optional[Dict] = None) -> MCPContext:
        """Create a new MCP context"""
        context = MCPContext(metadata=metadata or {})
        self.contexts[context.context_id] = context
        return context
    
    def get_context(self, context_id: str) -> Optional[MCPContext]:
        """Retrieve an existing context"""
        return self.contexts.get(context_id)
    
    def update_context(self, context_id: str, message: MCPMessage):
        """Update context with a new message"""
        context = self.get_context(context_id)
        if context:
            context.messages.append(message)
            context.updated_at = datetime.utcnow()
            
            # Trim context if it exceeds max size
            if len(context.messages) > self.max_context_size:
                context.messages = context.messages[-self.max_context_size:]
    
    def delete_context(self, context_id: str):
        """Delete a context"""
        if context_id in self.contexts:
            del self.contexts[context_id]
    
    def clean_expired_contexts(self):
        """Remove expired contexts based on TTL"""
        now = datetime.utcnow()
        expired = [
            ctx_id for ctx_id, ctx in self.contexts.items()
            if (now - ctx.updated_at).total_seconds() > self.context_ttl
        ]
        for ctx_id in expired:
            self.delete_context(ctx_id)
```

## Skills System

### Base Skill Interface

```python
# lightclaw/skills/base.py

from abc import ABC, abstractmethod
from typing import Dict, Any, Optional
from dataclasses import dataclass

@dataclass
class SkillResult:
    """Result from skill execution"""
    success: bool
    output: Any
    error: Optional[str] = None
    metadata: Optional[Dict[str, Any]] = None

class Skill(ABC):
    """Base class for all skills"""
    
    def __init__(self, name: str, description: str, version: str = "1.0.0"):
        self.name = name
        self.description = description
        self.version = version
        self.dependencies = []
    
    @abstractmethod
    async def execute(self, context: Dict[str, Any], params: Dict[str, Any]) -> SkillResult:
        """
        Execute the skill
        
        Args:
            context: MCP context and execution environment
            params: Skill-specific parameters
            
        Returns:
            SkillResult with execution outcome
        """
        pass
    
    def validate_params(self, params: Dict[str, Any]) -> bool:
        """Validate skill parameters"""
        return True
    
    async def on_load(self):
        """Called when skill is loaded"""
        pass
    
    async def on_unload(self):
        """Called when skill is unloaded"""
        pass
```

### Example: Slack Messaging Skill

```python
# lightclaw/skills/builtin/slack_messaging.py

from typing import Dict, Any
from ..base import Skill, SkillResult

class SlackMessagingSkill(Skill):
    """Skill for sending messages to Slack"""
    
    def __init__(self):
        super().__init__(
            name="slack_messaging",
            description="Send formatted messages to Slack channels",
            version="1.0.0"
        )
        self.slack_client = None
    
    async def on_load(self):
        """Initialize Slack client when skill loads"""
        from lightclaw.integrations.slack import SlackClient
        self.slack_client = SlackClient()
    
    async def execute(self, context: Dict[str, Any], params: Dict[str, Any]) -> SkillResult:
        """
        Send a message to Slack
        
        Params:
            channel: Target Slack channel
            text: Message text
            thread_ts: Optional thread timestamp for replies
            blocks: Optional rich formatting blocks
        """
        try:
            channel = params.get("channel")
            text = params.get("text")
            
            if not channel or not text:
                return SkillResult(
                    success=False,
                    output=None,
                    error="Missing required parameters: channel and text"
                )
            
            # Send message via Slack client
            response = await self.slack_client.post_message(
                channel=channel,
                text=text,
                thread_ts=params.get("thread_ts"),
                blocks=params.get("blocks")
            )
            
            return SkillResult(
                success=True,
                output=response,
                metadata={"message_ts": response.get("ts")}
            )
            
        except Exception as e:
            return SkillResult(
                success=False,
                output=None,
                error=str(e)
            )
```

## Slack Integration

### Slack Client Implementation

```python
# lightclaw/integrations/slack/client.py

import os
from typing import Dict, Any, Optional, List
from slack_sdk.web.async_client import AsyncWebClient
from slack_sdk.errors import SlackApiError

class SlackClient:
    """Async Slack client wrapper"""
    
    def __init__(self, bot_token: Optional[str] = None):
        self.bot_token = bot_token or os.getenv("SLACK_BOT_TOKEN")
        self.client = AsyncWebClient(token=self.bot_token)
    
    async def post_message(
        self,
        channel: str,
        text: str,
        thread_ts: Optional[str] = None,
        blocks: Optional[List[Dict]] = None
    ) -> Dict[str, Any]:
        """Post a message to a Slack channel"""
        try:
            response = await self.client.chat_postMessage(
                channel=channel,
                text=text,
                thread_ts=thread_ts,
                blocks=blocks
            )
            return response.data
        except SlackApiError as e:
            raise Exception(f"Slack API error: {e.response['error']}")
    
    async def get_channel_history(
        self,
        channel: str,
        limit: int = 100
    ) -> List[Dict[str, Any]]:
        """Get message history from a channel"""
        try:
            response = await self.client.conversations_history(
                channel=channel,
                limit=limit
            )
            return response.data["messages"]
        except SlackApiError as e:
            raise Exception(f"Slack API error: {e.response['error']}")
    
    async def add_reaction(self, channel: str, timestamp: str, emoji: str):
        """Add a reaction to a message"""
        try:
            await self.client.reactions_add(
                channel=channel,
                timestamp=timestamp,
                name=emoji
            )
        except SlackApiError as e:
            raise Exception(f"Slack API error: {e.response['error']}")
```

## Complete Example

### Main Agent Implementation

```python
# lightclaw/core/agent.py

import asyncio
from typing import Dict, Any, Optional, List
from dataclasses import dataclass

from .config import Config
from ..providers.provider_manager import ProviderManager
from ..mcp.protocol import MCPProtocol, MCPContext
from ..skills.engine import SkillsEngine

@dataclass
class AgentResponse:
    """Response from agent processing"""
    output: str
    context_id: str
    skills_used: List[str]
    metadata: Dict[str, Any]

class LightClawAgent:
    """Main LightClaw agent orchestrating all components"""
    
    def __init__(self, config: Config):
        self.config = config
        self.provider_manager = ProviderManager(config.providers)
        self.mcp = MCPProtocol(config.mcp)
        self.skills_engine = SkillsEngine(config.skills)
        self.slack_client = None
        
        if config.integrations.get("slack", {}).get("enabled"):
            from ..integrations.slack import SlackClient
            self.slack_client = SlackClient()
    
    async def initialize(self):
        """Initialize all components"""
        await self.provider_manager.initialize()
        await self.skills_engine.load_skills()
    
    async def process(
        self,
        prompt: str,
        context_id: Optional[str] = None,
        skills: Optional[List[str]] = None,
        metadata: Optional[Dict] = None
    ) -> AgentResponse:
        """
        Process a request through the agent
        
        Args:
            prompt: User input
            context_id: Optional existing context ID
            skills: List of skills to enable for this request
            metadata: Additional metadata
            
        Returns:
            AgentResponse with processing results
        """
        
        # Get or create MCP context
        if context_id:
            context = self.mcp.get_context(context_id)
            if not context:
                context = self.mcp.create_context(metadata)
        else:
            context = self.mcp.create_context(metadata)
        
        # Add user message to context
        context.add_message("user", prompt)
        
        # Determine which skills to use
        enabled_skills = skills or self.config.skills.get("allowed_skills", [])
        skills_used = []
        
        # Execute relevant skills
        skill_results = {}
        for skill_name in enabled_skills:
            skill = self.skills_engine.get_skill(skill_name)
            if skill and await self._should_use_skill(skill, prompt, context):
                result = await skill.execute(
                    context=context.to_dict(),
                    params={"prompt": prompt}
                )
                if result.success:
                    skill_results[skill_name] = result.output
                    skills_used.append(skill_name)
        
        # Prepare enhanced prompt with skill results
        enhanced_prompt = self._enhance_prompt(prompt, skill_results)
        
        # Get response from model provider
        provider = self.provider_manager.get_default_provider()
        model_response = await provider.generate(
            prompt=enhanced_prompt,
            context=context.to_dict()
        )
        
        # Add assistant response to context
        context.add_message("assistant", model_response.content)
        
        return AgentResponse(
            output=model_response.content,
            context_id=context.context_id,
            skills_used=skills_used,
            metadata={
                "tokens_used": model_response.tokens_used,
                "skill_results": skill_results
            }
        )
    
    async def _should_use_skill(
        self,
        skill,
        prompt: str,
        context: MCPContext
    ) -> bool:
        """Determine if a skill should be used for this request"""
        # Simple keyword-based detection (can be enhanced with ML)
        keywords = {
            "slack_messaging": ["send", "post", "message", "slack"],
            "web_search": ["search", "find", "look up", "google"],
            "code_analysis": ["analyze", "review", "code", "bug"]
        }
        
        skill_keywords = keywords.get(skill.name, [])
        prompt_lower = prompt.lower()
        
        return any(keyword in prompt_lower for keyword in skill_keywords)
    
    def _enhance_prompt(self, prompt: str, skill_results: Dict[str, Any]) -> str:
        """Enhance prompt with skill execution results"""
        if not skill_results:
            return prompt
        
        enhancement = "\n\nAdditional context from skills:\n"
        for skill_name, result in skill_results.items():
            enhancement += f"- {skill_name}: {result}\n"
        
        return prompt + enhancement
    
    async def close(self):
        """Cleanup resources"""
        await self.provider_manager.close()
        await self.skills_engine.unload_skills()
```

### Usage Example

```python
# example_usage.py

import asyncio
from lightclaw import LightClawAgent, Config

async def main():
    # Load configuration
    config = Config.from_file("config/lightclaw.yaml")
    
    # Initialize agent
    agent = LightClawAgent(config)
    await agent.initialize()
    
    try:
        # Process a request
        response = await agent.process(
            prompt="Can you search for the latest Python trends and post a summary to #general?",
            skills=["web_search", "slack_messaging"]
        )
        
        print(f"Response: {response.output}")
        print(f"Skills used: {response.skills_used}")
        print(f"Context ID: {response.context_id}")
        
        # Continue conversation with same context
        response2 = await agent.process(
            prompt="Can you elaborate on that?",
            context_id=response.context_id
        )
        
        print(f"\nFollow-up response: {response2.output}")
        
    finally:
        await agent.close()

if __name__ == "__main__":
    asyncio.run(main())
```

## Requirements File

```txt
# requirements.txt

# Core dependencies
aiohttp>=3.8.0
pyyaml>=6.0
python-dotenv>=0.19.0

# Google Cloud dependencies
google-auth>=2.0.0
google-auth-oauthlib>=0.4.0
google-auth-httplib2>=0.1.0

# Slack integration
slack-sdk>=3.19.0

# Optional dependencies
# For additional providers
anthropic>=0.3.0  # Optional: Anthropic Claude
openai>=0.27.0    # Optional: OpenAI GPT

# For development
pytest>=7.0.0
pytest-asyncio>=0.18.0
black>=22.0.0
mypy>=0.950
```

## Setup Script

```python
# setup.py

from setuptools import setup, find_packages

setup(
    name="lightclaw",
    version="1.0.0",
    description="Lightweight AI agent framework with MCP support",
    author="Your Name",
    author_email="your.email@example.com",
    packages=find_packages(),
    install_requires=[
        "aiohttp>=3.8.0",
        "pyyaml>=6.0",
        "python-dotenv>=0.19.0",
        "google-auth>=2.0.0",
        "google-auth-oauthlib>=0.4.0",
        "google-auth-httplib2>=0.1.0",
        "slack-sdk>=3.19.0",
    ],
    extras_require={
        "dev": [
            "pytest>=7.0.0",
            "pytest-asyncio>=0.18.0",
            "black>=22.0.0",
            "mypy>=0.950",
        ],
        "all-providers": [
            "anthropic>=0.3.0",
            "openai>=0.27.0",
        ],
    },
    entry_points={
        "console_scripts": [
            "lightclaw=lightclaw.cli:main",
        ],
    },
    python_requires=">=3.9",
    classifiers=[
        "Development Status :: 4 - Beta",
        "Intended Audience :: Developers",
        "License :: OSI Approved :: MIT License",
        "Programming Language :: Python :: 3.9",
        "Programming Language :: Python :: 3.10",
        "Programming Language :: Python :: 3.11",
    ],
)
```

## Testing

```python
# tests/test_agent.py

import pytest
from lightclaw import LightClawAgent, Config

@pytest.mark.asyncio
async def test_agent_initialization():
    """Test agent initialization"""
    config = Config.from_dict({
        "core": {"name": "TestAgent"},
        "providers": {},
        "mcp": {},
        "skills": {},
        "integrations": {}
    })
    
    agent = LightClawAgent(config)
    await agent.initialize()
    
    assert agent.config.core["name"] == "TestAgent"
    
    await agent.close()

@pytest.mark.asyncio
async def test_process_request():
    """Test processing a simple request"""
    config = Config.from_file("config/lightclaw.yaml")
    agent = LightClawAgent(config)
    await agent.initialize()
    
    try:
        response = await agent.process("Hello, how are you?")
        
        assert response.output is not None
        assert response.context_id is not None
        assert isinstance(response.skills_used, list)
    finally:
        await agent.close()
```

## Next Steps

1. Implement the core modules following this guide
2. Add comprehensive error handling and logging
3. Create extensive test coverage
4. Build CLI interface
5. Add REST API server
6. Create Docker images
7. Write detailed API documentation
8. Set up CI/CD pipeline

This implementation provides a solid foundation for a lightweight, extensible AI agent framework with all the requested features.
