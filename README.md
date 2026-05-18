# Hyperliquid Copy Trading Bot

## Overview

A production-grade copy trading bot built in TypeScript (Node.js) that mirrors trades from a target Hyperliquid wallet in real time. It subscribes to the target account’s activity over WebSocket, reacts to fills, and attempts to **replicate opens, adds, reductions, and closes** on your account while applying configurable risk limits.

### What it does

- Keeps a live connection to Hyperliquid and processes new fill events as they appear for `TARGET_WALLET`.
- Scales copied size using the equity ratio between your account and the target (with `SIZE_MULTIPLIER`), then enforces notional, leverage, blocked-asset, and concurrency limits.
- Supports **testnet** and **dry-run** so you can validate behavior before risking mainnet capital.
- Optional **Telegram** alerts and periodic **health checks** to compare your positions to the target’s snapshot.

### What it does *not* guarantee

- **Identical execution** to the target: latency, slippage, partial fills, Funding/cancel quirks, and API errors can cause divergence.
- **Profit** or **capital preservation**: copy trading inherits the target’s strategy *and* downside.
- **Compliance**: you are responsible for whether copying a given wallet is allowed in your jurisdiction and for the target’s terms of use.

### Who it’s for

Traders and builders who already understand perpetuals, leverage, and self-custody keys, and who want an automated “follow” layer with explicit risk caps—not a turnkey guaranteed system.

## Features

### Core Functionality

- **Real-time copying**: Monitors a target wallet (public or vault address) for new fills/trades via WebSocket subscriptions, and mirrors opens, reduces, closes of positions immediately.
- **Smart risk management & sizing**:
  - Position size = `(ourAccountEquity / targetWalletEquity) * targetPositionSize * SIZE_MULTIPLIER`
  - Configurable multiplier (e.g. 0.5×, 1×, 2×)
  - Minimum notional check (skip if too small) — Hyperliquid requires ~$10 per order
  - Maximum position size cap (as % of our equity)
  - Match leverage, but cap at MAX_LEVERAGE
  - Blocked assets support (skip certain assets)
  - Limit on max concurrent open trades

### Safety & Resilience

- **Dry-run / simulation mode**: Log actions without placing real orders
- **Testnet support**: Toggleable via config
- **Graceful reconnects**: Automatic WebSocket reconnection on disconnect with exponential backoff
- **Rate limiting**: Respects API limits with extra safety layer
- **Error handling & retries**: Automatic retry logic for failed orders

### Configuration & Validation

- `.env` + `zod` schema for strongly typed config
- Required: `PRIVATE_KEY`, `TARGET_WALLET`, `TESTNET` (boolean)
- Optional: `SIZE_MULTIPLIER`, `MAX_LEVERAGE`, `BLOCKED_ASSETS` (array), `DRY_RUN`, `LOG_LEVEL`
- `npm run validate` — loads config and checks `.env` / `node_modules` before you start (`scripts/validate-setup.ts`)

### Architecture & Code Quality

- Modern TypeScript (ESM, strict mode)
- Modular structure: separate modules for config, SDK init, monitoring, execution, logger, types, etc.
- Async/await, concurrency where necessary
- Comprehensive logging (info, warn, error) with timestamps via `winston`
- Full type safety using SDK types
- CLI script (e.g. `npm start`)
- Comments & notes on Hyperliquid specifics (e.g. no trailing zeros in size/price, GTC orders, etc.)

### Bonus Features

- **Telegram notifications**: Optional notifications when trades are copied (via `node-telegram-bot-api`)
- **Basic PnL tracking**: Compare our account vs target
- **Health checks**: Periodically verify our positions (every 5 minutes by default)

## Tech Stack & Dependencies

- **Node.js** (ESM), **TypeScript** with `strict` enabled
- **SDK**: `@nktkas/hyperliquid` (preferred community SDK)
- **Other libs**:
  - `dotenv` - Environment variable management
  - `zod` - Config schema validation
  - `winston` - Logging
  - `ethers` - Wallet management
  - `ws` - WebSocket support
  - Optional: `node-telegram-bot-api` - Telegram notifications

## Project Structure

```
Hyperliquid-copytrading-bot/
├── package.json
├── tsconfig.json
├── .env.example
├── README.md
├── scripts/
│   └── validate-setup.ts      # Pre-flight config check (npm run validate)
├── src/
│   ├── index.ts                 # Main entry point
│   ├── config.ts                # Configuration with zod validation
│   ├── hyperliquidClient.ts    # Hyperliquid SDK wrapper
│   ├── copyTrader.ts            # Core copy trading logic
│   ├── logger.ts                # Winston logger setup
│   ├── types.ts                 # TypeScript type definitions
│   ├── utils/
│   │   ├── risk.ts              # Risk management utilities
│   │   └── healthCheck.ts       # Health check utility
│   └── notifications/
│       └── telegram.ts          # Telegram notification service
└── logs/                        # Log files (auto-created)
```

## Setup & Installation

### Prerequisites

- Node.js 18+ (ESM support)
- npm or yarn
- Hyperliquid account (testnet or mainnet)

### Installation Steps

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd Hyperliquid-copytrading-bot
   ```

2. **Install dependencies**
   ```bash
   npm install
   ```

3. **Configure environment variables**
   ```bash
   cp .env.example .env
   ```
   
   Edit `.env` and fill in your values:
   ```env
   PRIVATE_KEY=0xYourPrivateKeyHere
   TARGET_WALLET=0xTargetWalletAddressHere
   TESTNET=true
   SIZE_MULTIPLIER=1.0
   MAX_LEVERAGE=20
   DRY_RUN=true
   ```

4. **Build TypeScript** (optional, can run directly with tsx)
   ```bash
   npm run build
   ```

5. **Validate configuration (recommended)**
   ```bash
   npm run validate
   ```
   Confirms `.env` loads, required variables parse, and basic paths exist before you go live.

6. **Start the bot**
   ```bash
   npm start
   ```

### Pre-flight checklist (mainnet)

1. Run on **testnet** with `DRY_RUN=true`, then `DRY_RUN=false`, before you trust mainnet behavior.
2. Confirm `TARGET_WALLET` is the **exact** address you intend to follow (vault vs EOA differences matter).
3. Set `MIN_NOTIONAL`, `MAX_LEVERAGE`, and `MAX_POSITION_SIZE_PERCENT` conservatively until you’ve observed a session.
4. Ensure the machine running the bot is **stable** (uptime, clock sync, network); WebSocket drops are handled but prolonged outages can desync positions vs the target.
5. **Never** commit `.env` or share `PRIVATE_KEY`; use a dedicated trading key with limited funds if possible.

## Configuration

### Required Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `PRIVATE_KEY` | Your wallet private key (0x prefix) | `0x1234...` |
| `TARGET_WALLET` | Target wallet address to copy | `0x5678...` |
| `TESTNET` | Use testnet (true/false) | `true` |

### Optional Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `SIZE_MULTIPLIER` | Position size multiplier | `1.0` |
| `MAX_LEVERAGE` | Maximum leverage cap | `20` |
| `MAX_POSITION_SIZE_PERCENT` | Max position size as % of equity | `50` |
| `MIN_NOTIONAL` | Minimum order size in USD | `10` |
| `MAX_CONCURRENT_TRADES` | Max concurrent open positions | `10` |
| `BLOCKED_ASSETS` | Comma-separated blocked assets | `` |
| `DRY_RUN` | Simulation mode (true/false) | `false` |
| `LOG_LEVEL` | Logging level (error/warn/info/debug) | `info` |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token (optional) | - |
| `TELEGRAM_CHAT_ID` | Telegram chat ID (optional) | - |
| `HEALTH_CHECK_INTERVAL` | Health check interval in minutes | `5` |

## Usage

### Basic Usage

1. **Testnet Testing** (Recommended first)
   ```env
   TESTNET=true
   DRY_RUN=true
   ```
   This allows you to test the bot without risking real funds.

2. **Production Mode**
   ```env
   TESTNET=false
   DRY_RUN=false
   ```

### How It Works

1. **Initialization**: Bot loads config, initializes Hyperliquid client, connects to your wallet
2. **Monitoring**: Subscribes to target wallet fills via WebSocket
3. **On Fill Event**:
   - Determines if it's opening, reducing, or closing a position
   - Fetches our and target wallet equity
   - Calculates position size: `(ourEquity / targetEquity) * targetSize * multiplier`
   - Applies risk checks (min notional, max size, blocked assets, leverage cap)
   - Executes trade (or logs in dry-run mode)
4. **Health Checks**: Periodically compares our positions vs target positions
5. **Notifications**: Sends Telegram notifications (if configured)

### Position Sizing Example

If:
- Target wallet equity: $10,000
- Our wallet equity: $5,000
- Target opens $1,000 position
- SIZE_MULTIPLIER: 1.0

Then our position size = `(5000 / 10000) * 1000 * 1.0 = $500`

### Risk Management

The bot includes multiple safety layers:

1. **Minimum Notional**: Skips trades below $10 (Hyperliquid requirement)
2. **Maximum Position Size**: Caps position at configured % of equity
3. **Leverage Cap**: Limits leverage to MAX_LEVERAGE
4. **Blocked Assets**: Skips copying certain coins
5. **Max Concurrent Trades**: Limits number of open positions
6. **Dry-Run Mode**: Test without placing real orders

## Hyperliquid SDK References

### SDK Compatibility Note


**Options**:
1. **@nktkas/hyperliquid** (preferred) - Community SDK
   - Install: `npm install @nktkas/hyperliquid`
   - GitHub: https://github.com/nktkas/hyperliquid

2. **nomeida/hyperliquid** (alternative) - Another community SDK
   - Check npm for availability

3. **Direct API calls** - If no SDK is available, implement direct HTTP/WebSocket calls
   - Official API docs: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api

**If SDK import fails**: The code includes error handling and will guide you. You may need to:
- Adjust import paths in `src/hyperliquidClient.ts`
- Implement direct API calls using `fetch` or `axios` for HTTP
- Use `ws` library for WebSocket connections

- **Official Hyperliquid Docs**: https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api

- **Testnet**:
  - Chain ID: 998
  - Faucet: https://app.hyperliquid-testnet.xyz/drip
  - Testnet Explorer: https://explorer.hyperliquid-testnet.xyz

### Important Hyperliquid Notes

- **No trailing zeros**: Size and price must not have trailing zeros (e.g., `"1.5"` not `"1.50"`)
- **GTC orders**: Orders are Good Till Cancel by default
- **Minimum notional**: ~$10 minimum per order
- **Leverage**: Must be set per coin before placing orders

## Logging & Monitoring

### Log Files

- `logs/combined.log` - All logs
- `logs/error.log` - Error logs only

### Log Levels

- `error` - Errors only
- `warn` - Warnings and errors
- `info` - Info, warnings, and errors (default)
- `debug` - Verbose logging

### Key Events Logged

- Subscription start/stop
- Detected fill events
- Trade opened/reduced/closed
- Skipped trades (due to filters)
- Errors & retries
- Health check results

## Telegram Notifications

The bot includes comprehensive Telegram notifications for real-time updates on trading activity, errors, and system status.

### Features

- **Trade Notifications**: Real-time alerts when trades are copied (success/failure)
- **Startup/Shutdown Alerts**: Notifications when bot starts or stops
- **Error Notifications**: Critical errors sent immediately with context
- **Health Check Alerts**: Warnings when position drift is detected
- **Summary Reports**: Optional daily/weekly trading statistics
- **Markdown Formatting**: Beautiful, readable messages with formatting

### Notification Types

1. **Trade Copied** - Sent when a trade is executed
   - Shows target trade details (coin, side, size, price)
   - Shows our trade details (side, size, leverage, order type)
   - Success/failure status with order ID or error message

2. **Startup Notification** - Sent when bot starts
   - Shows configuration summary
   - Testnet/dry-run status
   - Target wallet info

3. **Shutdown Notification** - Sent when bot stops gracefully

4. **Error Notifications** - Sent for critical errors
   - Error message and stack trace (in debug mode)
   - Context information

5. **Health Check Alerts** - Sent when position drift detected
   - Account equity comparison
   - Position drift details
   - Warning indicators

6. **Summary Reports** - Optional statistics
   - Total trades copied
   - Success rate
   - Active positions
   - PnL (if available)

### Setup Telegram Notifications

1. **Create a Telegram Bot**:
   - Open Telegram and search for [@BotFather](https://t.me/botfather)
   - Send `/newbot` command
   - Follow instructions to create your bot
   - Copy the bot token (e.g., `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`)

2. **Get Your Chat ID**:
   - Start a conversation with your bot
   - Send any message to the bot
   - Visit: `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`
   - Look for `"chat":{"id":123456789}` - that's your chat ID

3. **Configure in `.env`**:
   ```env
   TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrsTUVwxyz
   TELEGRAM_CHAT_ID=123456789
   ```

4. **Restart the bot** - Notifications will be enabled automatically

### Example Notifications

**Trade Copied:**
```
✅ Trade Copied

Target Trade:
• Coin: `BTC`
• Direction: Open Long
• Size: `0.1`
• Price: `50000`

Our Trade:
📈 Side: Long
• Size: `0.05`
• Leverage: `10x`
• Reduce Only: No
• Order Type: Market

Status: Success
• Order ID: `12345`

Time: 1/19/2026, 2:00:00 PM
```

**Health Check Alert:**
```
⚠️ Position drift

Account Status:
• Our Equity: $5000.00
• Target Equity: $10000.00
• Our Positions: 2
• Target Positions: 2

• BTC: Our 0.05 vs Target 0.1 (Diff: 0.05)

Checked: 1/19/2026, 2:00:00 PM
```

### Disabling Notifications

To disable Telegram notifications, simply remove or comment out the `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` from your `.env` file.

## Health Checks

The bot periodically checks positions:

- Compares our positions vs target positions
- Detects drift (size differences)
- Logs warnings if significant drift detected
- Default interval: 5 minutes (configurable)

## Troubleshooting

### Common Issues

1. **"Failed to import Hyperliquid SDK"**
   - Run: `npm install @nktkas/hyperliquid`
   - If package doesn't exist, check alternative SDKs or use official API

2. **"Cannot connect to Hyperliquid account"**
   - Verify `PRIVATE_KEY` is correct
   - Check network connectivity
   - Ensure testnet/mainnet matches your account

3. **"WebSocket disconnected"**
   - Bot will auto-reconnect with exponential backoff
   - Check network stability
   - Verify target wallet address is correct

4. **"Trade execution failed"**
   - Check account balance
   - Verify minimum notional requirements
   - Check if asset is blocked
   - Review leverage limits

### Debug Mode

Enable debug logging:
```env
LOG_LEVEL=debug
```

## Risks & Disclaimers

⚠️ **IMPORTANT WARNINGS**:

- **High Risk**: Trading derivatives involves significant risk of loss
- **Leverage Risk**: Leverage amplifies both gains and losses
- **Capital Loss**: You may lose your entire capital
- **No Warranty**: This software is provided "as is" without warranty
- **Educational Purpose**: This is for educational purposes only
- **Test First**: Always test on testnet before using real funds
- **Private Key Security**: Never share your private key. Store securely.
- **Operational Risk**: Bugs, exchange changes, or connectivity issues can cause missed trades or unintended exposure—monitor logs and account state continuously while running the bot.

## Contact

**Telegram**: [t.me/Kei4650](https://t.me/Kei4650) (@Kei4650)

Use this for questions about this project, integration help, or other requests. Response time is best-effort; for bugs, prefer opening a GitHub issue with logs and redacted config.

## Development

### Building

```bash
npm run build
```

### Development Mode

```bash
npm run dev
```

### Code Structure

- **Modular design**: Each component in separate file
- **Type safety**: Full TypeScript types throughout
- **Error handling**: Comprehensive try-catch blocks
- **Logging**: Structured logging at all levels

## Contributing

Contributions welcome! Please ensure:

- TypeScript strict mode compliance
- All tests pass
- Code is well-documented
- Follow existing code style

## License

MIT License — free to use, modify, but no warranty.

## Support

For issues, questions, or contributions:

1. **Contact**: [Telegram — @Kei4650](https://t.me/Kei4650) for direct questions about setup or behavior (see [Contact](#contact))
2. Check existing GitHub issues for bug reports and discussions
3. Review [Hyperliquid API docs](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api)
4. Reproduce on **testnet** first; attach relevant log excerpts (redact secrets)
5. Set `LOG_LEVEL=debug` when diagnosing execution or subscription issues

## Getting Started with Testnet

1. Visit https://app.hyperliquid-testnet.xyz/drip
2. Get testnet tokens from faucet
3. Set `TESTNET=true` in `.env`
4. Set `DRY_RUN=true` for initial testing
5. Start bot: `npm start`

## CLI Commands

```bash
npm start            # Start the bot (tsx src/index.ts)
npm run build        # Compile TypeScript (tsc)
npm run dev          # Development mode with file watch
npm run validate     # Validate .env and basic setup (scripts/validate-setup.ts)
npm run rebuild      # npm rebuild (native deps, if needed)
```

## Example .env File

```env
# Required
PRIVATE_KEY=0xYourPrivateKeyHere
TARGET_WALLET=0xTargetWalletAddressHere
TESTNET=true

# Position Sizing & Risk Management
SIZE_MULTIPLIER=1.0
MAX_LEVERAGE=20
MAX_POSITION_SIZE_PERCENT=50
MIN_NOTIONAL=10
MAX_CONCURRENT_TRADES=10

# Asset Filtering
BLOCKED_ASSETS=BTC,ETH

# Safety Features
DRY_RUN=true
LOG_LEVEL=info

# Optional: Telegram Notifications
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=

# Optional: Health Check Interval (minutes)
HEALTH_CHECK_INTERVAL=5
```

---

