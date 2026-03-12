# Installing React Native Space for Codex

Enable react-native-space skills in Codex via native skill discovery. Just clone and symlink.

## Prerequisites

- Git

## Installation

1. **Clone the react-native-space repository:**
   ```bash
   git clone https://github.com/react-native-vibe-code/react-native-space.git ~/.codex/react-native-space
   ```

2. **Create the skills symlink:**
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/react-native-space/skills ~/.agents/skills/react-native-space
   ```

   **Windows (PowerShell):**
   ```powershell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
   cmd /c mklink /J "$env:USERPROFILE\.agents\skills\react-native-space" "$env:USERPROFILE\.codex\react-native-space\skills"
   ```

3. **Restart Codex** (quit and relaunch the CLI) to discover the skills.

## Migrating from old bootstrap

If you installed react-native-space before native skill discovery, you need to:

1. **Update the repo:**
   ```bash
   cd ~/.codex/react-native-space && git pull
   ```

2. **Create the skills symlink** (step 2 above) — this is the new discovery mechanism.

3. **Remove the old bootstrap block** from `~/.codex/AGENTS.md` — any block referencing `react-native-space-codex bootstrap` is no longer needed.

4. **Restart Codex.**

## Verify

```bash
ls -la ~/.agents/skills/react-native-space
```

You should see a symlink (or junction on Windows) pointing to your react-native-space skills directory.

## Updating

```bash
cd ~/.codex/react-native-space && git pull
```

Skills update instantly through the symlink.

## Uninstalling

```bash
rm ~/.agents/skills/react-native-space
```

Optionally delete the clone: `rm -rf ~/.codex/react-native-space`.
