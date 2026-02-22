---
name: agent-browser
description: Web research automation using agent-browser - headless browser for AI agents with compact text output
version: 1.0.0
author: agentskills
tags: [web, research, browser, automation, scraping]
---

# Agent Browser Research Skill

Automate web research using [agent-browser](https://agent-browser.dev) - a headless browser designed for AI agents with token-efficient output.

## Installation

```bash
npm install -g agent-browser      # All platforms (native Rust CLI)
brew install agent-browser        # macOS
```

## Quick Start

```bash
# Navigate to a URL
agent-browser open https://example.com

# Get compact snapshot with element refs
agent-browser snapshot -i

# Interact with specific elements using refs
agent-browser click @e1
agent-browser type "search query" --target @e2
agent-browser submit @e3

# Take screenshots
agent-browser screenshot page.png

# Close browser
agent-browser close
```

## Core Commands for Research

### Navigation
- `open <url>` - Navigate to URL
- `back`, `forward` - Browser history
- `reload` - Refresh page
- `goto <url>` - Navigate without history

### Page Information
- `snapshot -i` - Get accessibility tree with refs
- `title` - Get page title
- `url` - Get current URL
- `wait [ms]` - Wait for ms or element

### Interaction
- `click @ref` - Click element by ref
- `type "text" --target @ref` - Type into input
- `clear @ref` - Clear input field
- `submit @ref` - Submit form
- `hover @ref` - Hover over element
- `scroll [px/@ref]` - Scroll page or to element

### Content Extraction
- `text @ref` - Get element text content
- `html @ref` - Get element HTML
- `screenshot [path]` - Capture screenshot
- `pdf [path]` - Save as PDF

### Search & Filters
- `find "selector"` - Find elements matching CSS selector
- `filter "text"` - Filter snapshot by text

## Research Workflow

```bash
# 1. Start research session
agent-browser open https://news.ycombinator.com

# 2. Capture page structure
agent-browser snapshot -i

# 3. Interact (example: click first story)
agent-browser click @e15

# 4. Extract content
agent-browser text @e8 > article.txt

# 5. Navigate back
agent-browser back

# 6. Search within page
agent-browser filter "AI"

# 7. Close when done
agent-browser close
```

## Best Practices

1. **Always use snapshot first** - Get element refs before interacting
2. **Use `-i` flag** - Include invisible elements in snapshot
3. **Save content to files** - Use redirection to preserve research
4. **Chain commands** - Use `&&` for sequential operations
5. **Take screenshots** - Visual confirmation of page state
6. **Use wait** - Wait for dynamic content: `agent-browser wait 2000`

## Common Patterns

```bash
# Research workflow
agent-browser open "https://www.google.com/search?q=AI+research" && \
  agent-browser snapshot -i && \
  agent-browser click @e25 && \
  agent-browser wait 1000 && \
  agent-browser text @e10 | tee results.txt

# Multi-page scraping
for url in $(cat urls.txt); do
  agent-browser open "$url"
  agent-browser text @e1 >> output.txt
  agent-browser screenshot "$(basename $url).png"
done

# Form submission
agent-browser open https://example.com/form
agent-browser type "query" --target @e1
agent-browser click @e2
agent-browser wait 2000
agent-browser snapshot -i
```

## Advanced Features

### Sessions
```bash
agent-browser session create research1
agent-browser open https://example.com --session research1
# ... do work ...
agent-browser session close research1
```

### Screenshots
```bash
agent-browser screenshot full.png      # Full page
agent-browser screenshot element.png --target @e5  # Specific element
agent-browser screenshot --full-page   # Capture entire scrollable page
```

### Network
```bash
agent-browser network log            # View network requests
agent-browser network clear          # Clear network log
```

## Tips

- **Refs are stable** within a session until page reloads
- **Use quotes** for URLs with special characters
- **Redirect output** to files for later analysis
- **Combine with grep** to filter results: `agent-browser text @e1 | grep "keyword"`
- **Use `--help`** for any command: `agent-browser click --help`

## Resources

- [Full Documentation](https://agent-browser.dev)
- [Commands Reference](https://agent-browser.dev/commands)
- [GitHub](https://github.com/vercel-labs/agent-browser)
