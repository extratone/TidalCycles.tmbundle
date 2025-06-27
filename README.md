## An attempt to use Google Gemini to code a TidalCycles bundle for TextMate.

---

_The following was genermated by LLM_

Of course. Based on the provided source code for the Visual Studio Code TidalCycles extension, here is a complete guide to creating an equivalent TextMate bundle (`.tmbundle`). This bundle will replicate the core functionality, including syntax highlighting, commands, and keybindings, adapted for the TextMate environment.

A TextMate bundle is a directory with a specific structure. You will need to create the following files and directories.

### Bundle Structure

First, create a directory named `TidalCycles.tmbundle`. All the following files will be placed inside it.

```
TidalCycles.tmbundle/
├── Commands/
│   ├── Evaluate Block.tmCommand
│   ├── Evaluate Line.tmCommand
│   ├── Hush.tmCommand
│   ├── Start Tidal Process.tmCommand
│   └── Stop Tidal Process.tmCommand
├── Syntaxes/
│   └── TidalCycles.tmLanguage
└── info.plist
```

-----

### 1\. `info.plist`

This is the main manifest file for the bundle. It defines the bundle's identity and its contents. Create a file named `info.plist` in the root of your bundle directory with the following content. A new UUID has been generated for this bundle.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>name</key>
    <string>TidalCycles</string>
    <key>uuid</key>
    <string>E4E3C9E2-7360-491A-AD6E-789E2B389A83</string>
    <key>ordering</key>
    <array>
        <string>E4E3C9E2-7360-491A-AD6E-789E2B389A83</string> <string>A0EFE414-A3D8-4554-B52A-60FF493D144A</string> <string>98D8055E-355D-47C7-947F-DCAD7980894C</string> <string>1E9F1A44-2454-4734-92B3-78F932152CD9</string> <string>63D9C3D6-5A04-4581-988B-D35070A67897</string> <string>3B1E4A34-521A-4B6A-B561-B9D8745E544A</string> </array>
    <key>mainMenu</key>
    <dict>
        <key>name</key>
        <string>TidalCycles</string>
        <key>items</key>
        <array>
            <string>A0EFE414-A3D8-4554-B52A-60FF493D144A</string> <string>98D8055E-355D-47C7-947F-DCAD7980894C</string> <string>1E9F1A44-2454-4734-92B3-78F932152CD9</string> <string>63D9C3D6-5A04-4581-988B-D35070A67897</string> <string>3B1E4A34-521A-4B6A-B561-B9D8745E544A</string> </array>
    </dict>
</dict>
</plist>
```

-----

### 2\. Syntax Highlighting

Create the directory `Syntaxes/`. Inside it, create the file `TidalCycles.tmLanguage`. This file defines the language grammar. It associates `.tidal` files with this grammar and cleverly includes the standard Haskell grammar, just as the VS Code extension recommends.

**File: `Syntaxes/TidalCycles.tmLanguage`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>fileTypes</key>
	<array>
		<string>tidal</string>
	</array>
	<key>name</key>
	<string>TidalCycles</string>
	<key>patterns</key>
	<array>
        <dict>
			<key>include</key>
			<string>source.haskell</string>
		</dict>
	</array>
	<key>scopeName</key>
	<string>source.tidal</string>
	<key>uuid</key>
	<string>F3540411-6677-495A-A656-62A7326D3F95</string>
</dict>
</plist>
```

*Note: This requires a Haskell TextMate bundle to be installed for full syntax highlighting.*

-----

### 3\. Commands

The core logic resides in commands. Each command is a `.tmCommand` file, which is an XML Plist defining metadata and a shell script. These commands manage a persistent `ghci` process using a named pipe for communication, which is a robust way to handle state between separate command executions.

Create the directory `Commands/` and add the following five files.

#### **File: `Start Tidal Process.tmCommand`**

This command starts the background `ghci` process and sends the boot sequence.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>name</key>
	<string>Start Tidal Process</string>
	<key>uuid</key>
	<string>A0EFE414-A3D8-4554-B52A-60FF493D144A</string>
	<key>scope</key>
	<string>source.tidal</string>
	<key>output</key>
	<string>showAsHTML</string>
	<key>command</key>
	<string>#!/bin/bash
# Get user-specific paths for the pipe and PID file
PIPE_PATH="/tmp/tidal_tm_pipe_$(id -u)"
PID_FILE="/tmp/tidal_tm_pid_$(id -u)"

# Check if process is already running
if [ -f "$PID_FILE" ]; then
    echo "&lt;h3&gt;Tidal Process Already Running&lt;/h3&gt;"
    echo "PID: $(cat $PID_FILE). Use 'Stop Tidal Process' to terminate it."
    exit 1
fi

# Create the named pipe (FIFO)
mkfifo "$PIPE_PATH"

# Determine which ghci command to use based on TextMate Variables
GHC_CMD="${TM_TIDAL_GHCIPATH:-ghci}"
if [[ "${TM_TIDAL_USE_STACK}" == "true" ]]; then
    GHC_CMD="stack ghci"
fi

echo "&lt;h3&gt;Starting Tidal...&lt;/h3&gt;"
echo "Using command: &lt;code&gt;$GHC_CMD&lt;/code&gt;&lt;br&gt;"

# Launch GHCi in the background, reading from our pipe
# Output is silenced for a cleaner experience, but can be logged if needed
eval "$GHC_CMD" &lt; "$PIPE_PATH" &gt; /dev/null 2&gt;&amp;1 &amp;

# Store the Process ID
echo $! &gt; "$PID_FILE"
echo "Process started with PID $(cat $PID_FILE).&lt;br&gt;"

# --- BOOT SEQUENCE ---
# Function to send a command to the running ghci process
function send_to_tidal() {
    echo "$1" &gt; "$PIPE_PATH"
}

# Use custom boot file if TM_TIDAL_BOOT_PATH is set
if [ -n "$TM_TIDAL_BOOT_PATH" ] &amp;&amp; [ -f "$TM_TIDAL_BOOT_PATH" ]; then
    echo "Loading boot file from: &lt;code&gt;$TM_TIDAL_BOOT_PATH&lt;/code&gt;"
    while IFS= read -r line; do
        send_to_tidal "$line"
    done &lt; "$TM_TIDAL_BOOT_PATH"
else
    # Default boot commands (from the VSCode extension source)
    echo "Loading default boot sequence."
    send_to_tidal ':set -XOverloadedStrings'
    send_to_tidal ':set prompt ""'
    send_to_tidal ':set prompt-cont ""'
    send_to_tidal 'import Sound.Tidal.Context'
    send_to_tidal 'tidal &lt;- startTidal (superdirtTarget {oLatency = 0.1, oAddress = "127.0.0.1", oPort = 57120}) (defaultConfig {cFrameTimespan = 1/20})'
    send_to_tidal 'let p = streamReplace tidal'
    send_to_tidal 'let hush = streamHush tidal'
    send_to_tidal 'let d1 = p 1'
    send_to_tidal 'let d2 = p 2'
    send_to_tidal 'let d3 = p 3'
    send_to_tidal 'let d4 = p 4'
    send_to_tidal 'let d5 = p 5'
    send_to_tidal 'let d6 = p 6'
    send_to_tidal 'let d7 = p 7'
    send_to_tidal 'let d8 = p 8'
    # ...add more default commands from src/tidal.ts if desired...
    send_to_tidal ':set prompt "tidal&gt; "'
fi

echo "&lt;h3&gt;Tidal Ready.&lt;/h3&gt;"
</string>
</dict>
</plist>
```

#### **File: `Evaluate Line.tmCommand`**

This command sends the current line to the running `ghci` process. The key binding is `Shift+Enter`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>name</key>
	<string>Evaluate Line</string>
	<key>uuid</key>
	<string>98D8055E-355D-47C7-947F-DCAD7980894C</string>
	<key>keyEquivalent</key>
	<string>~@\r</string> <key>scope</key>
	<string>source.tidal</string>
	<key>input</key>
	<string>line</string>
	<key>output</key>
	<string>discard</string>
	<key>command</key>
	<string>#!/bin/bash
PIPE_PATH="/tmp/tidal_tm_pipe_$(id -u)"
if [ ! -p "$PIPE_PATH" ]; then
    osascript -e 'display notification "Tidal process not running. Please start it first." with title "TidalCycles Error"'
    exit 1
fi

LINE_CONTENT="$TM_CURRENT_LINE"

# Send the expression wrapped in multiline markers
echo ":{" &gt; "$PIPE_PATH"
echo "$LINE_CONTENT" &gt; "$PIPE_PATH"
echo ":}" &gt; "$PIPE_PATH"
</string>
</dict>
</plist>
```

#### **File: `Evaluate Block.tmCommand`**

This command sends the selected text (or the entire document as a fallback) to `ghci`. The key binding is `Ctrl+Enter`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>name</key>
	<string>Evaluate Block</string>
	<key>uuid</key>
	<string>1E9F1A44-2454-4734-92B3-78F932152CD9</string>
	<key>keyEquivalent</key>
	<string>^\r</string> <key>scope</key>
	<string>source.tidal</string>
	<key>input</key>
	<string>selection</string>
	<key>fallbackInput</key>
    <string>document</string>
	<key>output</key>
	<string>discard</string>
	<key>command</key>
	<string>#!/bin/bash
PIPE_PATH="/tmp/tidal_tm_pipe_$(id -u)"
if [ ! -p "$PIPE_PATH" ]; then
    osascript -e 'display notification "Tidal process not running. Please start it first." with title "TidalCycles Error"'
    exit 1
fi

# Read the entire input (selection or document) from stdin
BLOCK_CONTENT=$(cat)

# Send the expression wrapped in multiline markers, line by line
echo ":{" &gt; "$PIPE_PATH"
echo "$BLOCK_CONTENT" | while IFS= read -r line; do
    echo "$line" &gt; "$PIPE_PATH"
done
echo ":}" &gt; "$PIPE_PATH"
</string>
</dict>
</plist>
```

#### **File: `Hush.tmCommand`**

Sends the `hush` command to stop all sound. The key binding is `Ctrl+Alt+H`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>name</key>
	<string>Hush</string>
	<key>uuid</key>
	<string>63D9C3D6-5A04-4581-988B-D35070A67897</string>
	<key>keyEquivalent</key>
	<string>^@H</string> <key>scope</key>
	<string>source.tidal</string>
	<key>output</key>
	<string>discard</string>
	<key>command</key>
	<string>#!/bin/bash
PIPE_PATH="/tmp/tidal_tm_pipe_$(id -u)"
if [ ! -p "$PIPE_PATH" ]; then
    osascript -e 'display notification "Tidal process not running. Please start it first." with title "TidalCycles Error"'
    exit 1
fi

# The hush command is also sent inside a block
echo ":{" &gt; "$PIPE_PATH"
echo "hush" &gt; "$PIPE_PATH"
echo ":}" &gt; "$PIPE_PATH"
</string>
</dict>
</plist>
```

#### **File: `Stop Tidal Process.tmCommand`**

This command safely terminates the background `ghci` process and cleans up the pipe and PID file.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>name</key>
	<string>Stop Tidal Process</string>
	<key>uuid</key>
	<string>3B1E4A34-521A-4B6A-B561-B9D8745E544A</string>
	<key>scope</key>
	<string>source.tidal</string>
	<key>output</key>
	<string>showAsTooltip</string>
	<key>command</key>
	<string>#!/bin/bash
PIPE_PATH="/tmp/tidal_tm_pipe_$(id -u)"
PID_FILE="/tmp/tidal_tm_pid_$(id -u)"

if [ -f "$PID_FILE" ]; then
    PID=$(cat "$PID_FILE")
    kill "$PID"
    rm "$PID_FILE"
    rm "$PIPE_PATH"
    echo "Tidal process $PID stopped."
else
    echo "Tidal process not found."
fi
</string>
</dict>
</plist>
```

### How to Use the Bundle

1.  **Installation**: Save all the files with the structure described above. Ensure the top-level folder is named `TidalCycles.tmbundle`. Double-click this folder, and TextMate will install it.
2.  **Prerequisites**: Install a [Haskell TextMate bundle](https://www.google.com/search?q=https://github.com/textmate/haskell-tmbundle) for syntax highlighting to work.
3.  **Configuration (Optional)**:
      * Open TextMate's Preferences and go to the "Variables" tab.
      * To use `stack`, add a variable named `TM_TIDAL_USE_STACK` and set its value to `true`.
      * To specify a path to `ghci`, add `TM_TIDAL_GHCIPATH` and set its value (e.g., `/opt/homebrew/bin/ghci`).
      * To use a custom boot file, add `TM_TIDAL_BOOT_PATH` and set its value to the absolute path of your boot file.
4.  **Workflow**:
      * Open a `.tidal` file.
      * Go to the `Bundles` -\> `TidalCycles` menu and select `Start Tidal Process`.
      * Use `Shift+Enter` to evaluate the current line or `Ctrl+Enter` to evaluate a selected block of code.
      * Use `Ctrl+Alt+H` to hush the audio.
      * When you are finished, select `Stop Tidal Process` from the menu.