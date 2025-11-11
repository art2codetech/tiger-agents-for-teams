# Microsoft Teams Bot Refactoring Plan for Tiger Agent

## Executive Summary

**YES, it is possible to adapt Tiger Agent to work with Microsoft Teams bots.**

However, this requires significant architectural changes due to fundamental differences between the Slack and Microsoft Teams bot platforms. This document outlines the technical differences, required refactoring, and implementation strategy.

### Key Findings

- **Core Architecture is Reusable**: The PostgreSQL-backed event queue, worker pool system, and AI agent logic (Pydantic-AI + MCP) can be preserved
- **Major Refactoring Required**: Complete replacement of Slack SDK integration with Microsoft 365 Agents SDK (~40-50% of codebase)
- **SDK Choice**: **M365 Agents SDK** is the recommended path (Bot Framework's official successor, supports multi-channel deployment)
- **Connection Model**: Shift from Slack Socket Mode to webhook endpoints with JWT authentication
- **Perfect AI Fit**: M365 Agents SDK is "unopinionated about AI" - works seamlessly with Tiger Agent's existing Pydantic-AI + MCP architecture

### Recommendation

**Dual-Platform Support with M365 Agents SDK**: Refactor into a platform-agnostic core with pluggable platform adapters (Slack and Teams), using M365 Agents SDK for Teams integration. This provides the best long-term foundation and multi-channel capabilities.

---

## Why M365 Agents SDK?

**UPDATED RECOMMENDATION** (based on latest SDK research):

After evaluating the SDK landscape, **Microsoft 365 Agents SDK is the clear choice** for Teams integration:

### Decision Matrix: Teams AI vs M365 Agents SDK

| Factor | Teams AI Library | M365 Agents SDK | Winner |
|--------|------------------|-----------------|---------|
| **Python Status** | Developer preview | ✅ Fully available | M365 SDK |
| **AI Integration** | Opinionated (built-in) | ✅ Unopinionated (BYO AI) | M365 SDK |
| **Tiger Agent Fit** | ❌ Conflicts with Pydantic-AI | ✅ Perfect fit | M365 SDK |
| **Channels Supported** | Teams only | ✅ 15+ channels | M365 SDK |
| **Long-term Support** | Unclear relationship to M365 SDK | ✅ Bot Framework successor | M365 SDK |
| **Bot Framework Migration** | N/A | ✅ Official migration path | M365 SDK |

### Why M365 Agents SDK is Perfect for Tiger Agent

1. **Unopinionated About AI** 🎯
   - Tiger Agent already has Pydantic-AI + MCP for orchestration
   - M365 SDK doesn't force its own AI framework
   - Teams AI Library would conflict with existing architecture

2. **Bot Framework Successor** 🏆
   - Official evolution/replacement for Bot Framework (deprecated Dec 2025)
   - Maintains compatibility during transition
   - Microsoft's long-term supported path

3. **Multi-Channel Ready** 🌐
   - Supports Teams, Slack, web chat, and 15+ other channels
   - Future-proof for additional platform support
   - Single SDK for all channels (if desired)

4. **Production Ready** ✅
   - Python SDK is fully available (not preview)
   - Active development and Microsoft support
   - Official documentation and samples available

5. **Modern Architecture** 🚀
   - Built on async/await patterns
   - aiohttp integration for webhook handling
   - Clean separation of concerns

---

## Platform Comparison

### 1. Connection Architecture

| Aspect | Slack | Microsoft Teams |
|--------|-------|-----------------|
| **SDK** | Slack Bolt SDK (Python) | M365 Agents SDK (Python) - Bot Framework successor |
| **Connection Method** | Socket Mode (WebSocket) | Webhook endpoint (HTTPS) with JWT auth |
| **Public Endpoint** | Not required (Socket Mode) | Required for webhook |
| **Behind Firewall** | Works natively | Requires public endpoint or Azure hosting |
| **Multi-Channel** | Slack-only | M365 Agents SDK supports 15+ channels (Teams, Slack, web chat, etc.) |
| **AI Integration** | No built-in AI | Unopinionated - bring your own AI framework |

### 2. Event Model

| Aspect | Slack | Microsoft Teams |
|--------|-------|-----------------|
| **Event Type** | `app_mention`, `message` events | `Activity` objects (message, conversationUpdate, invoke, etc.) |
| **Event Schema** | JSON with event_ts, text, user, channel, etc. | M365 Agents Activity schema with from, conversation, text, etc. |
| **Message ID** | `ts` (timestamp string) | `id` (unique activity ID) |
| **User ID** | Slack user ID (U1234567) | AAD object ID or channel-specific ID |
| **Channel ID** | Slack channel ID (C1234567) | Conversation ID (channel, 1:1, group chat) |
| **Threading** | `thread_ts` field | `replyToId` field |
| **@Mentions** | Parsed from text with `<@U1234567>` | Structured in `entities` with mention objects |

### 3. API Interaction

| Aspect | Slack | Microsoft Teams |
|--------|-------|-----------------|
| **SDK Client** | `AsyncWebClient` | `TurnContext` + `CloudAdapter` |
| **Send Message** | `chat_postMessage` | `TurnContext.send_activity` |
| **Reactions** | `reactions_add/remove` | Not directly supported (use typing indicators or messages) |
| **User Info** | `users_info` API | Microsoft Graph API (optional) |
| **Bot Info** | `auth_test` + `bots_info` | Provided in `Activity.recipient` |
| **Handler Pattern** | Event listeners | `TeamsActivityHandler` subclass |

### 4. Authentication

| Aspect | Slack | Microsoft Teams |
|--------|-------|-----------------|
| **Bot Token** | `xoxb-...` (Bearer token) | Azure AD app credentials (App ID + Password/Certificate) |
| **App Token** | `xapp-...` (for Socket Mode) | Not applicable (JWT validation on webhook) |
| **Scopes** | OAuth scopes (users:read, chat:write, etc.) | M365 Agents authentication + Graph API permissions (optional) |
| **Validation** | Token-based authentication | JWT signature verification via CloudAdapter |

---

## Current Architecture Analysis

### Components that Can Be Preserved

1. **EventHarness** (`tiger_agent/harness.py`)
   - Worker pool system with bounded concurrency
   - Event claiming and retry logic
   - Database integration
   - ✅ **Platform-agnostic** - only needs event insertion and processing callbacks

2. **Database Layer** (`tiger_agent/migrations/`)
   - PostgreSQL + TimescaleDB event queue
   - Migration system
   - ✅ **Platform-agnostic** - stores generic JSONB events

3. **TigerAgent Core** (`tiger_agent/agent.py`)
   - Pydantic-AI integration
   - MCP server support
   - Jinja2 prompt templating
   - ✅ **Mostly platform-agnostic** - only context generation needs adaptation

4. **Utilities**
   - Logging configuration
   - Migration runner
   - ✅ **Platform-agnostic**

### Components Requiring Replacement

1. **Slack Integration** (`tiger_agent/slack.py`) - **100% replacement**
   - All Slack API functions
   - Slack data models (UserInfo, BotInfo, ChannelInfo)
   - Reaction management

2. **Event Handling** (`tiger_agent/harness.py`) - **Partial refactoring**
   - `_on_event` callback registration (Slack Bolt → M365 Agents handler)
   - Socket Mode handler → Webhook endpoint with aiohttp server
   - Slack-specific event filters

3. **Data Models** (`tiger_agent/types.py`) - **Extension needed**
   - Current models are Slack-specific
   - Need platform-agnostic Event abstraction

4. **CLI Entry Point** (`tiger_agent/main.py`) - **Minor changes**
   - Configuration for platform selection
   - Different initialization for Teams vs Slack

---

## Refactoring Strategy

### Phase 1: Abstract Platform Layer

**Goal**: Separate platform-specific code from business logic

#### 1.1 Create Platform Abstraction

Create new module `tiger_agent/platforms/base.py`:

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Any

@dataclass
class PlatformMessage:
    """Platform-agnostic message representation"""
    id: str
    timestamp: datetime
    text: str
    user_id: str
    user_name: str | None
    channel_id: str
    channel_name: str | None
    thread_id: str | None
    raw_event: dict[str, Any]

@dataclass
class PlatformUser:
    """Platform-agnostic user representation"""
    id: str
    name: str
    display_name: str | None
    email: str | None
    timezone: str | None

@dataclass
class PlatformBot:
    """Platform-agnostic bot representation"""
    id: str
    name: str
    app_id: str

class PlatformAdapter(ABC):
    """Abstract base class for platform integrations"""

    @abstractmethod
    async def start(self, event_callback: Callable) -> None:
        """Start listening for events"""
        pass

    @abstractmethod
    async def send_message(self, channel_id: str, text: str, thread_id: str | None = None) -> None:
        """Send a message to the platform"""
        pass

    @abstractmethod
    async def add_reaction(self, channel_id: str, message_id: str, emoji: str) -> None:
        """Add a reaction (if supported by platform)"""
        pass

    @abstractmethod
    async def get_user_info(self, user_id: str) -> PlatformUser | None:
        """Fetch user information"""
        pass

    @abstractmethod
    async def get_bot_info(self) -> PlatformBot:
        """Get bot identity"""
        pass

    @abstractmethod
    def parse_event(self, raw_event: dict[str, Any]) -> PlatformMessage:
        """Parse platform-specific event into standard format"""
        pass
```

#### 1.2 Refactor Slack Integration

Move existing Slack code to `tiger_agent/platforms/slack.py`:

```python
class SlackAdapter(PlatformAdapter):
    """Slack platform adapter using Bolt SDK"""

    def __init__(self, bot_token: str, app_token: str):
        self.app = AsyncApp(token=bot_token)
        self.app_token = app_token
        self.client = self.app.client

    async def start(self, event_callback: Callable) -> None:
        """Start Slack Socket Mode connection"""
        async def on_event(ack: AsyncAck, event: dict):
            await ack()
            message = self.parse_event(event)
            await event_callback(message)

        self.app.event("app_mention")(on_event)
        handler = AsyncSocketModeHandler(self.app, self.app_token)
        await handler.start_async()

    def parse_event(self, event: dict) -> PlatformMessage:
        """Convert Slack event to PlatformMessage"""
        return PlatformMessage(
            id=event["ts"],
            timestamp=datetime.fromtimestamp(float(event["event_ts"])),
            text=event["text"],
            user_id=event["user"],
            user_name=None,  # Fetch separately if needed
            channel_id=event["channel"],
            channel_name=None,
            thread_id=event.get("thread_ts"),
            raw_event=event
        )

    async def send_message(self, channel_id: str, text: str, thread_id: str | None = None) -> None:
        """Send Slack message"""
        await self.client.chat_postMessage(
            channel=channel_id,
            thread_ts=thread_id,
            text=text,
            blocks=[{"type": "markdown", "text": text}]
        )

    # ... implement other methods
```

#### 1.3 Update EventHarness

Refactor `EventHarness` to accept a `PlatformAdapter`:

```python
class EventHarness:
    def __init__(
        self,
        event_processor: EventProcessor,
        platform: PlatformAdapter,
        pool: AsyncConnectionPool | None = None,
        # ... other params
    ):
        self._platform = platform
        self._event_processor = event_processor
        # ... rest of initialization

    async def run(self):
        """Start the event harness"""
        await self._pool.open(wait=True)

        async with asyncio.TaskGroup() as tasks:
            # Run migrations
            async with self._pool.connection() as con:
                await runner.migrate_db(con)

            # Start workers
            for worker_id, initial_sleep in self._worker_args(self._num_workers):
                tasks.create_task(self._worker(worker_id, initial_sleep))

            # Start platform connection
            tasks.create_task(self._platform.start(self._on_event))
```

### Phase 2: Implement Teams Adapter

#### 2.1 Create Teams Module Structure

```
tiger_agent/platforms/teams/
├── __init__.py
├── adapter.py          # TeamsAdapter implementation
├── bot.py              # Bot Framework bot class
├── models.py           # Teams-specific data models
└── auth.py             # Azure AD authentication helpers
```

#### 2.2 Implement TeamsAdapter with M365 Agents SDK

Create `tiger_agent/platforms/teams/bot.py`:

```python
from microsoft_agents.hosting.core import TurnContext
from microsoft_agents.hosting.teams import TeamsActivityHandler
from microsoft_agents.activity import Activity, ActivityTypes
from typing import Callable

class TeamsBot(TeamsActivityHandler):
    """Teams bot using M365 Agents SDK"""

    def __init__(self, event_callback: Callable):
        super().__init__()
        self.event_callback = event_callback
        self.conversation_refs = {}  # Store for sending proactive messages

    async def on_message_activity(self, turn_context: TurnContext):
        """Handle incoming message activities"""
        activity = turn_context.activity

        # Store conversation reference for later use
        self._store_conversation_ref(turn_context)

        # Check if bot was mentioned (for channel messages)
        if activity.conversation.conversation_type == "channel":
            if not self._is_bot_mentioned(activity):
                return

        # Convert to platform-agnostic message
        message = self._parse_activity(activity)

        # Trigger event processing
        if self.event_callback:
            await self.event_callback(message)

    def _is_bot_mentioned(self, activity: Activity) -> bool:
        """Check if bot was @mentioned"""
        if not activity.entities:
            return False

        return any(
            e.type == "mention" and
            e.mentioned.id == activity.recipient.id
            for e in activity.entities
        )

    def _store_conversation_ref(self, turn_context: TurnContext):
        """Store conversation reference for proactive messaging"""
        from microsoft_agents.activity import TurnContextExtensions
        ref = TurnContextExtensions.get_conversation_reference(turn_context.activity)
        self.conversation_refs[turn_context.activity.conversation.id] = ref

    def _parse_activity(self, activity: Activity) -> dict:
        """Convert Activity to platform-agnostic dict"""
        return {
            "id": activity.id,
            "timestamp": activity.timestamp,
            "text": activity.text or "",
            "user_id": activity.from_property.id,
            "user_name": activity.from_property.name,
            "channel_id": activity.conversation.id,
            "channel_name": getattr(activity.conversation, "name", None),
            "thread_id": activity.reply_to_id,
            "raw_event": activity
        }
```

Create `tiger_agent/platforms/teams/adapter.py`:

```python
from microsoft_agents.hosting.core import CloudAdapter
from microsoft_agents.hosting.aiohttp import start_agent_process
from microsoft_agents.activity import Activity, ActivityTypes
from tiger_agent.platforms.base import PlatformAdapter, PlatformMessage, PlatformUser, PlatformBot
from tiger_agent.platforms.teams.bot import TeamsBot
from datetime import datetime
from typing import Callable

class TeamsAdapter(PlatformAdapter):
    """Microsoft Teams platform adapter using M365 Agents SDK"""

    def __init__(self, app_id: str, app_password: str, port: int = 3978):
        self.app_id = app_id
        self.app_password = app_password
        self.port = port
        self.bot = None
        self.adapter = None

    async def start(self, event_callback: Callable) -> None:
        """Start M365 Agents SDK webhook server"""
        # Create bot with event callback
        self.bot = TeamsBot(event_callback)

        # Create CloudAdapter with Azure AD authentication
        self.adapter = CloudAdapter(
            app_id=self.app_id,
            app_password=self.app_password
        )

        # Start aiohttp server on /api/messages endpoint
        # This is provided by M365 Agents SDK
        await start_agent_process(
            bot=self.bot,
            adapter=self.adapter,
            port=self.port
        )

        print(f"Teams bot listening on http://0.0.0.0:{self.port}/api/messages")

    def parse_event(self, event: dict | Activity) -> PlatformMessage:
        """Convert Teams Activity to PlatformMessage"""
        if isinstance(event, dict):
            activity = event.get("raw_event", event)
        else:
            activity = event

        return PlatformMessage(
            id=activity.id if hasattr(activity, "id") else event.get("id"),
            timestamp=activity.timestamp if hasattr(activity, "timestamp") else datetime.now(),
            text=event.get("text", ""),
            user_id=event.get("user_id", ""),
            user_name=event.get("user_name"),
            channel_id=event.get("channel_id", ""),
            channel_name=event.get("channel_name"),
            thread_id=event.get("thread_id"),
            raw_event=event
        )

    async def send_message(self, channel_id: str, text: str, thread_id: str | None = None) -> None:
        """Send Teams message using stored conversation reference"""
        # Get stored conversation reference
        conv_ref = self.bot.conversation_refs.get(channel_id)

        if not conv_ref:
            raise ValueError(f"No conversation reference found for channel {channel_id}")

        # Create activity callback
        async def send_callback(turn_context: TurnContext):
            reply = Activity(
                type=ActivityTypes.message,
                text=text
            )

            if thread_id:
                reply.reply_to_id = thread_id

            await turn_context.send_activity(reply)

        # Send using adapter's continue_conversation
        await self.adapter.continue_conversation(
            conv_ref,
            send_callback,
            audience=self.app_id
        )

    async def add_reaction(self, channel_id: str, message_id: str, emoji: str) -> None:
        """Teams doesn't support reactions like Slack - use typing indicator instead"""
        # Teams doesn't have direct reaction API
        # Instead, we send a typing indicator
        await self.send_typing_indicator(channel_id)

    async def send_typing_indicator(self, channel_id: str) -> None:
        """Send typing indicator to show bot is processing"""
        conv_ref = self.bot.conversation_refs.get(channel_id)

        if conv_ref:
            async def typing_callback(turn_context: TurnContext):
                await turn_context.send_activity(
                    Activity(type=ActivityTypes.typing)
                )

            await self.adapter.continue_conversation(
                conv_ref,
                typing_callback,
                audience=self.app_id
            )

    async def get_user_info(self, user_id: str) -> PlatformUser | None:
        """Fetch Teams user info (basic from activity, optionally enhance with Graph API)"""
        # Basic implementation returns minimal info
        # Can be enhanced with Microsoft Graph API integration
        return PlatformUser(
            id=user_id,
            name="Unknown",
            display_name=None,
            email=None,
            timezone=None
        )

    async def get_bot_info(self) -> PlatformBot:
        """Get Teams bot identity"""
        return PlatformBot(
            id=self.app_id,
            name="Tiger Agent",
            app_id=self.app_id
        )
```

#### 2.3 Update Configuration

Add Teams configuration to `.env`:

```bash
# Platform Selection
PLATFORM=teams  # or "slack"

# Teams Configuration
TEAMS_APP_ID=<azure-app-id>
TEAMS_APP_PASSWORD=<azure-app-password>
TEAMS_BOT_PORT=3978

# Slack Configuration (existing)
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
```

#### 2.4 Update CLI Entry Point

Modify `tiger_agent/main.py`:

```python
@cli.command()
@click.option("--platform", type=click.Choice(["slack", "teams"]), default="slack", help="Chat platform to use")
@click.option("--teams-app-id", help="Teams App ID (for Teams platform)")
@click.option("--teams-app-password", help="Teams App Password (for Teams platform)")
# ... other options
def run(platform: str, teams_app_id: str | None = None, teams_app_password: str | None = None, ...):
    """Run the Tiger Agent bot"""
    load_dotenv(dotenv_path=env if env else find_dotenv(usecwd=True))
    setup_logging()

    # Create platform adapter based on selection
    if platform == "slack":
        from tiger_agent.platforms.slack import SlackAdapter
        adapter = SlackAdapter(
            bot_token=os.getenv("SLACK_BOT_TOKEN"),
            app_token=os.getenv("SLACK_APP_TOKEN")
        )
    elif platform == "teams":
        from tiger_agent.platforms.teams import TeamsAdapter
        adapter = TeamsAdapter(
            app_id=teams_app_id or os.getenv("TEAMS_APP_ID"),
            app_password=teams_app_password or os.getenv("TEAMS_APP_PASSWORD"),
            port=int(os.getenv("TEAMS_BOT_PORT", "3978"))
        )
    else:
        raise ValueError(f"Unsupported platform: {platform}")

    # Build agent
    agent = TigerAgent(model=model, ...)

    # Create harness with platform adapter
    harness = EventHarness(agent, platform=adapter, ...)

    # Run
    asyncio.run(harness.run())
```

### Phase 3: Handle Platform-Specific Features

#### 3.1 Reaction Handling

**Problem**: Slack reactions (`spinthinking`, `white_check_mark`, `x`) don't exist in Teams

**Solutions**:
1. **Typing Indicators**: Use Teams typing indicators during processing
2. **Status Messages**: Post/edit messages like "Processing..." → "Complete"
3. **Adaptive Cards**: Use Teams Adaptive Cards with status indicators
4. **No-op**: Make reactions optional in the core agent logic

**Recommended Approach**: Make reactions adapter-specific

```python
# In TigerAgent
async def __call__(self, hctx: HarnessContext, event: Event) -> None:
    platform = hctx.platform
    mention = event.event

    try:
        # Platform-agnostic: Signal processing started
        await platform.signal_processing_started(mention.channel, mention.id)

        response = await self.generate_response(hctx, event)
        await platform.send_message(mention.channel, response, mention.thread_id)

        # Platform-agnostic: Signal success
        await platform.signal_processing_complete(mention.channel, mention.id, success=True)
    except Exception as e:
        await platform.signal_processing_complete(mention.channel, mention.id, success=False)
        raise
```

#### 3.2 User Context

**Problem**: Slack provides rich user context (timezone, profile), Teams requires Microsoft Graph

**Solutions**:
1. **Basic Profile**: Use info from Activity object (name, ID)
2. **Graph API Integration**: Optional enhanced user lookup via Microsoft Graph
3. **Cached Context**: Store user info in database on first interaction

#### 3.3 Threading Model

**Both platforms support threading**:
- Slack: `thread_ts` field
- Teams: `replyToId` field

The `PlatformMessage.thread_id` abstraction handles this.

### Phase 4: Database Schema Updates

#### 4.1 Platform Field

The current schema stores raw JSONB events. **No schema changes required** since events are already stored as generic JSON.

Optional enhancement:

```sql
-- Add platform tracking (optional)
ALTER TABLE agent.event ADD COLUMN platform text DEFAULT 'slack';
CREATE INDEX idx_event_platform ON agent.event(platform);
```

#### 4.2 Message ID Mapping

Slack uses timestamp-based IDs (`ts`), Teams uses GUIDs. This is already handled by storing raw events.

### Phase 5: Testing Strategy

#### 5.1 Unit Tests

- Test each platform adapter independently
- Mock Bot Framework adapter for Teams tests
- Mock Slack Bolt app for Slack tests

#### 5.2 Integration Tests

- Test EventHarness with mock platform adapters
- Verify event claiming and processing works with both platforms
- Test database operations remain platform-agnostic

#### 5.3 Manual Testing

**Slack**:
1. Deploy with `--platform slack`
2. Test @mentions in channels
3. Test DMs
4. Verify reactions and threading

**Teams**:
1. Create Azure Bot Service resource
2. Configure webhook endpoint: `https://<your-domain>/api/messages`
3. Deploy with `--platform teams`
4. Test @mentions in channels
5. Test 1:1 chats
6. Verify threading (replies)

---

## Challenges and Risks

### 1. ~~SDK Deprecation~~ ✅ RESOLVED

**Previous Challenge**: Bot Framework SDK support ends December 31, 2025

**Resolution**: **Using M365 Agents SDK** - the official successor to Bot Framework, fully supported and future-proof.

**Benefits of M365 Agents SDK**:
- ✅ Long-term Microsoft support (Bot Framework successor)
- ✅ Unopinionated AI integration (perfect for Pydantic-AI + MCP)
- ✅ Multi-channel support (Teams, Slack, web chat, and 15+ others)
- ✅ Python SDK fully available (not preview)
- ✅ Built on modern async patterns
- ✅ Official migration path from Bot Framework

**Status**: **Low Risk** - M365 Agents SDK is the recommended, supported path forward.

### 2. Webhook Endpoint Requirement

**Challenge**: Teams requires public HTTPS endpoint (unlike Slack Socket Mode)

**Impact**: Medium - deployment complexity increases

**Mitigation**:
- Use Azure Bot Service (simplest)
- Use ngrok/localtunnel for development
- Deploy behind reverse proxy (nginx + Let's Encrypt)
- Consider Azure Functions or AWS Lambda for serverless deployment

### 3. Missing Features

| Feature | Slack | Teams | Mitigation |
|---------|-------|-------|------------|
| Reactions | Native | Not available | Use typing indicators or status messages |
| Socket Mode | Native | Not available | Use webhook endpoint |
| Rich user context | Native | Requires Graph API | Basic context from Activity, optional Graph integration |
| File sharing | Native | Native but different API | Platform-specific implementations |

### 4. Authentication Complexity

**Challenge**: Azure AD authentication is more complex than Slack tokens

**Impact**: Medium - setup and configuration more involved

**Mitigation**:
- Provide detailed setup documentation
- Create Azure ARM/Bicep templates for bot provisioning
- Consider managed identity for Azure deployments

### 5. Message Format Differences

**Challenge**: Teams and Slack have different markdown and formatting capabilities

**Impact**: Low - mostly cosmetic

**Mitigation**:
- Use lowest common denominator (basic markdown)
- Platform-specific formatting in adapters
- Test AI responses render correctly on both platforms

---

## Implementation Roadmap

### Milestone 1: Platform Abstraction (2-3 weeks)
- [ ] Create `PlatformAdapter` abstract base class
- [ ] Define platform-agnostic data models
- [ ] Refactor Slack code into `SlackAdapter`
- [ ] Update `EventHarness` to use platform adapters
- [ ] Update `TigerAgent` to use platform-agnostic models
- [ ] Update CLI for platform selection
- [ ] Add unit tests for Slack adapter
- [ ] Verify Slack functionality unchanged

### Milestone 2: Teams Adapter Implementation (3-4 weeks)
- [ ] Set up M365 Agents SDK dependencies
- [ ] Implement `TeamsBot` with `TeamsActivityHandler`
- [ ] Implement `TeamsAdapter` with CloudAdapter and aiohttp
- [ ] Implement activity parsing and message sending
- [ ] Add conversation reference management for proactive messaging
- [ ] Create Azure Bot Service setup documentation
- [ ] Implement typing indicators for "processing" state
- [ ] Add unit tests for Teams adapter
- [ ] Local testing with ngrok

### Milestone 3: Platform-Specific Features (1-2 weeks)
- [ ] Implement Slack reactions
- [ ] Implement Teams typing indicators
- [ ] Add optional Microsoft Graph integration for user context
- [ ] Handle platform-specific formatting
- [ ] Test thread handling on both platforms
- [ ] Document feature parity differences

### Milestone 4: Documentation and Deployment (1 week)
- [ ] Update README with Teams setup instructions
- [ ] Create Azure deployment guide
- [ ] Add environment variable reference
- [ ] Create Teams app manifest template
- [ ] Add troubleshooting guide
- [ ] Update CLAUDE.md

### Milestone 5: Testing and Validation (1-2 weeks)
- [ ] Integration testing with real Teams tenant
- [ ] Load testing with both platforms
- [ ] Security review of webhook endpoint
- [ ] Performance benchmarking
- [ ] User acceptance testing
- [ ] Bug fixes and polish

**Total Estimated Timeline**: 8-12 weeks for full implementation

---

## Code Organization

### Proposed Directory Structure

```
tiger_agent/
├── __init__.py
├── agent.py                    # Platform-agnostic TigerAgent
├── harness.py                  # Platform-agnostic EventHarness
├── types.py                    # Shared types
├── main.py                     # CLI with platform selection
├── platforms/
│   ├── __init__.py
│   ├── base.py                 # PlatformAdapter + base models
│   ├── slack/
│   │   ├── __init__.py
│   │   ├── adapter.py          # SlackAdapter
│   │   └── models.py           # Slack-specific models
│   └── teams/
│       ├── __init__.py
│       ├── adapter.py          # TeamsAdapter
│       ├── bot.py              # Bot Framework bot class
│       ├── models.py           # Teams-specific models
│       └── auth.py             # Azure AD helpers
├── migrations/                 # No changes needed
└── commands.py                 # No changes needed
```

### Dependencies Update

`pyproject.toml` additions:

```toml
[project]
dependencies = [
    # Existing
    "pydantic>=2.0",
    "pydantic-ai",
    "psycopg[binary]",
    "psycopg-pool",
    "logfire",

    # Slack (existing)
    "slack-bolt>=1.18",
    "slack-sdk>=3.0",

    # Teams - M365 Agents SDK (new)
    "microsoft-agents-hosting-core",
    "microsoft-agents-hosting-aiohttp",
    "microsoft-agents-hosting-extensions-teams",
    "microsoft-agents-activity",
    "microsoft-agents-authentication",

    # Optional: Microsoft Graph for enhanced user context
    "msgraph-sdk>=1.0",
]
```

**Note**: M365 Agents SDK requires Python 3.10+ (Python 3.11+ recommended for optimal performance).

---

## Alternative Approaches

### Option 1: Separate Repositories

**Approach**: Fork Tiger Agent into separate `tiger-agent-slack` and `tiger-agent-teams` repos

**Pros**:
- Simpler codebases
- No platform abstraction overhead
- Independent release cycles

**Cons**:
- Code duplication
- Maintenance burden
- Feature parity challenges
- Separate documentation

**Verdict**: ❌ Not recommended - abstraction overhead is worth unified codebase

### Option 2: M365 Agents SDK as Universal Adapter

**Approach**: Use M365 Agents SDK for both Slack AND Teams (SDK supports 15+ channels)

**Pros**:
- Single integration model
- M365 SDK handles platform differences
- Built-in multi-channel support (Teams, Slack, web chat, etc.)
- Future-proof (Bot Framework successor)
- Unopinionated about AI (works with Pydantic-AI)

**Cons**:
- M365 SDK Slack support may be limited compared to native Slack Bolt
- Loses Slack-specific features (Socket Mode, native reactions, etc.)
- More complex for Slack-only users
- Additional abstraction layer

**Verdict**: ⚠️ Interesting option for future - but start with native Slack SDK for best Slack experience, use M365 SDK for Teams only

### Option 3: Multi-Platform from Day 1

**Approach**: Support Slack, Teams, Discord, Telegram, etc. via unified adapter pattern

**Pros**:
- Maximum flexibility
- Broader market appeal
- Robust abstraction layer

**Cons**:
- Scope creep
- Testing complexity
- Feature matrix gets complicated
- Longer time to market

**Verdict**: ⚠️ Future consideration - start with Slack + Teams, design for extensibility

---

## Migration Path for Existing Users

### For Current Slack Users

**No breaking changes required**:

```bash
# Existing usage continues to work
uv run tiger_agent run

# Explicit platform selection (optional)
uv run tiger_agent run --platform slack
```

### For New Teams Users

```bash
# Install with Teams support
uv sync

# Configure environment
export PLATFORM=teams
export TEAMS_APP_ID=<your-app-id>
export TEAMS_APP_PASSWORD=<your-app-password>

# Run with Teams
uv run tiger_agent run --platform teams
```

### Hybrid Deployments

Run multiple instances with shared database:

```bash
# Instance 1: Slack
uv run tiger_agent run --platform slack --num-workers 3

# Instance 2: Teams
uv run tiger_agent run --platform teams --num-workers 3
```

Both instances share the same PostgreSQL event queue and can process events from either platform.

---

## Security Considerations

### 1. Webhook Endpoint Security

**Slack**: Socket Mode doesn't expose public endpoint

**Teams**: Webhook requires:
- HTTPS with valid TLS certificate
- JWT signature verification (handled by M365 Agents SDK)
- Request validation against Azure AD

**Implementation**:

```python
class TeamsAdapter:
    async def start(self, event_callback: Callable) -> None:
        # CloudAdapter automatically handles JWT validation
        self.adapter = CloudAdapter(
            app_id=self.app_id,
            app_password=self.app_password
        )

        # start_agent_process handles webhook setup with built-in security
        await start_agent_process(
            bot=self.bot,
            adapter=self.adapter,  # JWT validation happens here
            port=self.port
        )
```

**Note**: M365 Agents SDK's `CloudAdapter` automatically validates JWT tokens from Azure Bot Service, ensuring only authenticated requests are processed.

### 2. Credential Management

**Slack**: Two tokens (bot token, app token)

**Teams**: App ID + Password/Certificate

**Best Practices**:
- Use environment variables (never commit tokens)
- Use Azure Key Vault for Teams credentials in production
- Rotate credentials regularly
- Use managed identities when possible (Azure deployments)

### 3. Data Privacy

**Consideration**: Events stored in database may contain sensitive information

**Recommendations**:
- Encrypt database at rest
- Implement data retention policies
- Add PII redaction option
- Document data handling in privacy policy

---

## Performance Implications

### Event Processing

**No significant performance impact**:
- Platform abstraction adds negligible overhead (<1ms per event)
- Database layer unchanged
- Worker pool system unchanged

### Connection Overhead

**Slack Socket Mode**: Persistent WebSocket connection (minimal overhead)

**Teams Webhook**: HTTP request per event (slightly higher overhead)

**Mitigation**: The worker pool and event queue architecture already handles high-throughput scenarios efficiently.

### Database Load

**No change**: Same PostgreSQL-backed queue regardless of platform

---

## Questions for Decision Making

Before starting implementation, clarify:

1. **Primary Use Case**:
   - Replace Slack with Teams?
   - Support both platforms simultaneously?
   - Prioritize one platform over the other?

2. **Deployment Model**:
   - Self-hosted on-premises?
   - Cloud-hosted (Azure, AWS, GCP)?
   - Hybrid (Slack self-hosted, Teams on Azure)?

3. **Feature Priorities**:
   - Must reactions work on Teams? (requires workaround)
   - Is Microsoft Graph integration required for user context?
   - Should adaptive cards be supported?

4. **Timeline**:
   - Deadline for Teams support?
   - Can we phase the rollout?
   - Is immediate Bot Framework OK knowing Dec 2025 deprecation?

5. **Resource Availability**:
   - Azure subscription available?
   - Can team set up Azure Bot Service?
   - Who will maintain Teams-specific code?

---

## Conclusion

**Microsoft Teams support is absolutely feasible** for Tiger Agent with moderate refactoring effort. The core architecture (event queue, worker pool, AI agent) is platform-agnostic and can be preserved.

### Recommended Path Forward

1. **Phase 1 (Weeks 1-3)**: Implement platform abstraction layer
2. **Phase 2 (Weeks 4-7)**: Build Teams adapter with **M365 Agents SDK**
3. **Phase 3 (Weeks 8-10)**: Handle platform-specific features
4. **Phase 4 (Weeks 11-12)**: Testing and documentation

### Key Success Factors

- ✅ Use M365 Agents SDK (Bot Framework's official successor)
- ✅ Use adapter pattern for clean platform separation
- ✅ Maintain backwards compatibility for Slack users
- ✅ Document setup clearly for both platforms
- ✅ Leverage M365 SDK's unopinionated AI approach for Pydantic-AI integration
- ✅ Keep event queue and worker system platform-agnostic
- ✅ Future-proof with multi-channel support built-in

### Next Steps

1. Get stakeholder approval on approach
2. Set up Azure Bot Service test environment
3. Create spike/proof-of-concept with Teams adapter
4. Validate webhook architecture works with EventHarness
5. Begin Milestone 1 implementation

---

## Appendix: Useful Resources

### Microsoft 365 Agents SDK (Primary)

- **[M365 Agents SDK Overview](https://learn.microsoft.com/en-us/microsoft-365/agents-sdk/agents-sdk-overview)** - Official SDK documentation
- **[M365 Agents SDK Python Quickstart](https://learn.microsoft.com/en-us/microsoft-365/agents-sdk/quickstart-python)** - Getting started guide
- **[M365 Agents SDK Python API Reference](https://learn.microsoft.com/en-us/python/api/agent-sdk-python/agents-overview)** - Complete API docs
- **[GitHub: microsoft/Agents-for-python](https://github.com/microsoft/Agents-for-python)** - Official Python SDK repository
- **[GitHub: microsoft/Agents](https://github.com/microsoft/Agents)** - Main repo with samples
- **[Bot Framework to M365 SDK Migration Guide](https://learn.microsoft.com/en-us/microsoft-365/agents-sdk/bf-migration-guidance)** - Official migration docs

### Microsoft Teams Bot Development

- [Teams Bot Quickstart (Python)](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/build-a-bot)
- [Teams Activity Handler Reference](https://microsoft.github.io/Agents-for-js/classes/_microsoft_agents-hosting-extensions-teams.TeamsActivityHandler.html)
- [Create and Deploy Custom Engine Agents](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/create-deploy-agents-sdk)

### Azure Bot Service

- [Create Azure Bot Resource](https://learn.microsoft.com/en-us/azure/bot-service/bot-service-quickstart-registration)
- [Configure Teams Channel](https://learn.microsoft.com/en-us/azure/bot-service/channel-connect-teams)

### Microsoft Graph API

- [Graph SDK for Python](https://github.com/microsoftgraph/msgraph-sdk-python)
- [Get User Info](https://learn.microsoft.com/en-us/graph/api/user-get)

### SDK Comparison & Migration

- [Teams SDK Evolution 2025](https://www.voitanos.io/blog/microsoft-teams-sdk-evolution-2025/) - Detailed SDK comparison
- [Why M365 Agents SDK Should Be on Your Radar](https://www.koskila.net/Why-m365-agents-sdk-should-be-on-your-radar/)
- [Microsoft Teams vs Slack](https://kinsta.com/blog/microsoft-teams-vs-slack/)
