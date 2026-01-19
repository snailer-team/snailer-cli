# @felixaihub/snailer

> AI-Powered Development Agent for Your Terminal

[![npm version](https://badge.fury.io/js/%40felixaihub%2Fsnailer.svg)](https://www.npmjs.com/package/@felixaihub/snailer)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## What is Snailer?

Snailer is an intelligent AI coding agent that lives in your terminal. This npm package provides a convenient cross-platform installer for the Snailer CLI.

```bash
# Just describe what you want in plain English
snailer --prompt "refactor the authentication module to use async/await"
snailer --prompt "find all TODO comments and create GitHub issues"
```

## Quick Install

### Global Installation

```bash
npm install -g @felixaihub/snailer
```

### Using npx (No Installation)

```bash
npx @felixaihub/snailer@latest --help
```

## Features

- 🧠 **Self-Learning**: Improves with every interaction using ACE (Agentic Context Engineering)
- 🔄 **Multi-Model Support**: Works with Claude, GPT, Grok, and Gemini
- 🛠️ **Built-in Tools**: File operations, code search, git workflows, and more
- ⚡ **Fast & Reliable**: Built with Rust for performance

## Setup

After installation, set up your AI API key:

```bash
# For Claude (recommended)
export CLAUDE_API_KEY="your-api-key"

# Or for OpenAI
export OPENAI_API_KEY="your-api-key"

# Or for xAI
export XAI_API_KEY="your-api-key"
```

## Usage

```bash
# Start interactive mode (REPL)
snailer

# Execute a specific task (one-shot)
snailer --prompt "Add error handling to the API module"

# Get help
snailer --help
```

## How It Works

This npm package is an installer wrapper that:

1. Downloads the appropriate Snailer binary from [GitHub Releases](https://github.com/felixaihub/snailer-cli/releases)
2. Selects the correct binary for your platform (macOS, Linux, Windows)
3. Installs it in a location accessible via the `snailer` command

### Supported Platforms

- **macOS**: x64, ARM64 (Apple Silicon)
- **Linux**: x64, ARM64
- **Windows**: x64

### Binary Download

During `npm install`, the package automatically:
- Detects your platform and architecture
- Downloads the corresponding binary from GitHub Releases
- Makes it executable and available in your PATH

### Offline Installation

To skip the automatic download (useful for air-gapped environments):

```bash
SNAILER_SKIP_POSTINSTALL=1 npm install -g @felixaihub/snailer
```

Then manually download the binary from [Releases](https://github.com/felixaihub/snailer-cli/releases) and place it in the package's `bin/` directory.

## Documentation

For detailed documentation, visit the main repository:

- [GitHub Repository](https://github.com/felixaihub/snailer-cli)
- [Architecture Overview](https://github.com/felixaihub/snailer-cli/blob/main/ARCHITECTURE.md)
- [Agent Documentation](https://github.com/felixaihub/snailer-cli/blob/main/docs/AGENT_ARCHITECTURE.md)
- [ACE System](https://github.com/felixaihub/snailer-cli/blob/main/docs/ACE_SYSTEM.md)
- [Contributing Guide](https://github.com/felixaihub/snailer-cli/blob/main/docs/CONTRIBUTING.md)

## Troubleshooting

### Binary download fails

If the automatic download fails, you can:

1. Manually download from [Releases](https://github.com/felixaihub/snailer-cli/releases)
2. Place the binary in the package installation directory under `bin/`
3. Make it executable: `chmod +x bin/snailer` (Unix-like systems)

### Command not found

If `snailer` command is not found after global installation:

1. Check that npm's global bin directory is in your PATH:
   ```bash
   npm config get prefix
   ```
2. Add it to your PATH if needed (usually `~/.npm/bin` or `/usr/local/bin`)

### Permission denied

On Unix-like systems, if you get "permission denied":

```bash
chmod +x $(which snailer)
```

## License

This installer package is licensed under the MIT License.

The Snailer binary is subject to its own [End User License Agreement (EULA)](https://github.com/felixaihub/snailer-cli/blob/main/EULA.md).

By installing and using Snailer, you agree to the terms in both licenses.

## Support

- **Issues**: [GitHub Issues](https://github.com/felixaihub/snailer-cli/issues)
- **Website**: [snailer.ai](https://snailer.ai)

---

**Made with ❤️ by the Snailer Team**
