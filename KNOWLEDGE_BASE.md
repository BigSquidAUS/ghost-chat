# Knowledge Base

This document records significant technical hurdles, crashes, and architecture resolutions encountered during the development of Ghost Chat.

---

## 1. Post-Reboot Windows Startup Failure (Silent Crash)

### Symptom
On Windows, after a computer reboot or sudden power loss, Ghost Chat would completely fail to open again. The application would crash silently on launch, showing no window, taskbar icon, or system tray presence. The only known workaround was to delete all Ghost Chat folders out of the `/AppData/Roaming` directory.

### Root Causes

1. **Non-Atomic Config Saving:**
   When the system reboots or shuts down, Windows terminates running processes. Ghost Chat registers standard shutdown hooks via Wails (`ShouldQuit`/`ServiceShutdown`) to save the last-known window position and size. This triggers a write to the configuration file using `os.WriteFile` on `%APPDATA%\ghost-chat\config.json`. If Windows terminates the process before the OS flushes the buffer to disk, or mid-write, `config.json` is left corrupted or containing 0 bytes.

2. **Invisible Startup Crash:**
   On the next launch, `config.Load()` attempts to parse the configuration file. When it encounters corrupted or empty JSON, it returns an error. The entrypoint `main.go` intercepts this error, calls `println` to log it, and terminates immediately. Because the application is compiled as a GUI-only binary (`-H windowsgui`), `stdout` and `stderr` are completely suppressed, causing the app to crash silently without any error prompt or UI indicator.

3. **Suboptimal WebView2 UDF Location:**
   By default, Wails v3 initializes WebView2's User Data Folder (UDF) in `%APPDATA%\[BinaryName.exe]`, resulting in `%APPDATA%\ghost-chat.exe`. Microsoft strongly advises against storing WebView2 user data in the Roaming profile due to its potential large size, volatility, and profile-syncing lock contention across shutdowns and boots.

---

### Solution

We implemented a three-tier self-healing and defensive saving solution to ensure the application always boots and writes state reliably:

#### A. Atomic Config Saving (`internal/config/store.go`)
Instead of overwriting `config.json` directly, we write the data to a temporary file, flush it physically to the storage hardware, and perform an OS-level rename:
1. Write JSON bytes to `config.json.tmp`.
2. Call `Sync()` on the temporary file descriptor to guarantee all bytes are written to physical disk.
3. Close the file.
4. Delete the existing `config.json` (required on Windows since Go's `os.Rename` does not overwrite existing files).
5. Rename `config.json.tmp` to `config.json`.

```go
func Save(config *Config, path string) error {
	bytes, err := json.MarshalIndent(config, "", "  ")
	if err != nil {
		return err
	}

	dir := filepath.Dir(path)
	if err = os.MkdirAll(dir, 0755); err != nil {
		return err
	}

	tempPath := path + ".tmp"
	f, err := os.OpenFile(tempPath, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0644)
	if err != nil {
		return err
	}
	defer func() {
		f.Close()
		_ = os.Remove(tempPath)
	}()

	if _, err = f.Write(bytes); err != nil {
		return err
	}

	if err = f.Sync(); err != nil {
		return err
	}

	if err = f.Close(); err != nil {
		return err
	}

	if err = os.Remove(path); err != nil && !errors.Is(err, os.ErrNotExist) {
		return err
	}

	return os.Rename(tempPath, path)
}
```

#### B. Self-Healing Config Loading (`internal/config/store.go`)
If the config file is corrupted or unparsable on boot, we recover gracefully rather than crashing:
1. When `json.Unmarshal` returns an error, log a warning.
2. Back up the corrupted file to `config.json.corrupted` so users can manually inspect/recover custom settings if desired.
3. Remove the corrupted `config.json` file.
4. Fallback to `DefaultConfig()` and return a valid configuration struct so the application boots successfully.

```go
func Load(path string) (*Config, error) {
	bytes, err := os.ReadFile(path)
	if errors.Is(err, os.ErrNotExist) {
		defaultConfig := DefaultConfig()
		return &defaultConfig, nil
	}
	if err != nil {
		return nil, err
	}

	var raw map[string]json.RawMessage
	if err = json.Unmarshal(bytes, &raw); err != nil {
		println("Warning: config file is corrupted, backing up and starting fresh:", err.Error())
		_ = os.WriteFile(path+".corrupted", bytes, 0644)
		_ = os.Remove(path)

		defaultConfig := DefaultConfig()
		return &defaultConfig, nil
	}

	if _, hasV3Key := raw["savedWindowState"]; hasV3Key {
		defaultConfig := DefaultConfig()
		return &defaultConfig, nil
	}

	config := DefaultConfig()
	if err = json.Unmarshal(bytes, &config); err != nil {
		println("Warning: config file could not be parsed, backing up and starting fresh:", err.Error())
		_ = os.WriteFile(path+".corrupted", bytes, 0644)
		_ = os.Remove(path)

		defaultConfig := DefaultConfig()
		return &defaultConfig, nil
	}

	return &config, nil
}
```

#### C. Relocating WebView2 User Data Folder (`main.go`)
We set `WebviewUserDataPath` in `application.Options.Windows` to point to a local, machine-specific directory instead of the Roaming folder:
1. Dynamically read `%LOCALAPPDATA%` on Windows.
2. Resolve `WebviewUserDataPath` as `%LOCALAPPDATA%\ghost-chat\webview`.
3. This isolates browser states, keeps caches off the network profile, and prevents locked folders or sync conflicts.

```go
	var webviewUserDataPath string

	if runtime.GOOS == "windows" {
		if localAppData := os.Getenv("LOCALAPPDATA"); localAppData != "" {
			webviewUserDataPath = filepath.Join(localAppData, "ghost-chat", "webview")
		}
	}

	app := application.New(application.Options{
		Name: "Ghost Chat",
		Icon: appIcon,
		Windows: application.WindowsOptions{
			WebviewUserDataPath: webviewUserDataPath,
		},
        // ...
    })

---

## 2. Grey Background Box In Vanish/Transparency Mode

### Symptom
When toggling Vanish mode (which makes the chat overlay transparent and click-through), the application background remains a solid grey box on Windows instead of becoming fully transparent and invisible.

### Root Cause
While Wails v3 supports transparent windows, the underlying WebView2 browser control on Windows defaults to rendering a solid background color (typically grey or white) when HTML components have their backgrounds set to transparent.
Although the Go side was configured with `BackgroundType: application.BackgroundTypeTransparent`, the window's `BackgroundColour` field was not specified. WebView2 defaults to a solid fall-back color under transparent HTML elements in the DOM unless explicitly instructed to render with an alpha value of `0` at the OS window layout level.

### Solution
We configured `BackgroundColour: application.NewRGBA(0, 0, 0, 0)` in `application.WebviewWindowOptions` within `main.go`. This instructs WebView2 to render transparent areas with a fully transparent backdrop (0 alpha), allowing the underlying desktop or other windows to show through.

```go
	win := app.Window.NewWithOptions(application.WebviewWindowOptions{
		Title:            "ghost-chat",
		Width:            initialW,
		Height:           initialH,
		Hidden:           true,
		Frameless:        true,
		AlwaysOnTop:      true,
		BackgroundType:   application.BackgroundTypeTransparent,
		BackgroundColour: application.NewRGBA(0, 0, 0, 0),
		Mac: application.MacWindow{
			Backdrop: application.MacBackdropTransparent,
		},
	})
``````
