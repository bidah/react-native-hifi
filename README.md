# React Native Space

React Native Space is a complete React Native & Expo development workflow for your coding agents, built on top of a set of composable "skills" and initial instructions that make sure your agent uses them.

Inspired by and based on [Superpowers](https://github.com/obra/superpowers) by Jesse Vincent.

## How it works

It starts from the moment you fire up your coding agent. As soon as it sees that you're building something, it *doesn't* just jump into trying to write code. Instead, it steps back and asks you what you're really trying to do.

Once it's teased a spec out of the conversation, it shows it to you in chunks short enough to actually read and digest.

After you've signed off on the design, your agent puts together an implementation plan that's clear enough for an enthusiastic junior engineer with poor taste, no judgement, no project context, and an aversion to testing to follow. It emphasizes true red/green TDD with React Native Testing Library, YAGNI (You Aren't Gonna Need It), and DRY.

Next up, once you say "go", it launches a *subagent-driven-development* process, having agents work through each engineering task, inspecting and reviewing their work, and continuing forward. It's not uncommon for Claude to be able to work autonomously for a couple hours at a time without deviating from the plan you put together.

On top of the core workflow, React Native Space includes specialized skills for Expo development: building native UI with SwiftUI and Jetpack Compose, setting up Tailwind CSS, configuring dev clients, creating API routes, fetching data natively, and testing on real devices through the agent-device skill.

## Installation

**Note:** Installation differs by platform. Claude Code or Cursor have built-in plugin marketplaces. Codex and OpenCode require manual setup.

### Claude Code (via Plugin Marketplace)

In Claude Code, register the marketplace first:

```bash
/plugin marketplace add rofi-rofi/react-native-space-marketplace
```

Then install the plugin from this marketplace:

```bash
/plugin install react-native-space@react-native-space-marketplace
```

### Cursor (via Plugin Marketplace)

In Cursor Agent chat, install from marketplace:

```text
/add-plugin react-native-space
```

or search for "react-native-space" in the plugin marketplace.

### Codex

Tell Codex:

```
Fetch and follow instructions from https://raw.githubusercontent.com/rofi-rofi/react-native-space/refs/heads/main/.codex/INSTALL.md
```

**Detailed docs:** [docs/README.codex.md](docs/README.codex.md)

### OpenCode

Tell OpenCode:

```
Fetch and follow instructions from https://raw.githubusercontent.com/rofi-rofi/react-native-space/refs/heads/main/.opencode/INSTALL.md
```

**Detailed docs:** [docs/README.opencode.md](docs/README.opencode.md)

### Gemini CLI

```bash
gemini extensions install https://github.com/rofi-rofi/react-native-space
```

To update:

```bash
gemini extensions update react-native-space
```

### Verify Installation

Start a new session in your chosen platform and ask for something that should trigger a skill (for example, "help me plan this feature" or "let's build a new screen"). The agent should automatically invoke the relevant react-native-space skill.

## Skills Overview

### Core Workflow

- **brainstorming** - Socratic design refinement before writing any code
- **writing-plans** - Detailed implementation plans broken into bite-sized tasks
- **executing-plans** - Batch execution with human checkpoints
- **subagent-driven-development** - Fast iteration with two-stage review (spec compliance, then code quality)
- **dispatching-parallel-agents** - Concurrent subagent workflows
- **using-git-worktrees** - Parallel development branches
- **finishing-a-development-branch** - Merge/PR decision workflow

### Testing & Debugging

- **test-driven-development** - RED-GREEN-REFACTOR cycle with React Native Testing Library (includes testing anti-patterns reference)
- **systematic-debugging** - 4-phase root cause process (includes root-cause-tracing, defense-in-depth, condition-based-waiting techniques)
- **verification-before-completion** - Ensure it's actually fixed

### Code Review

- **requesting-code-review** - Pre-review checklist
- **receiving-code-review** - Responding to feedback

### React Native & Expo

- **building-native-ui** - Build native UI components using platform APIs
- **expo-api-routes** - Create and manage Expo API routes
- **expo-dev-client** - Configure and use Expo dev clients for native module development
- **expo-tailwind-setup** - Set up Tailwind CSS (NativeWind) in Expo projects
- **expo-ui-jetpack-compose** - Build native Android UI with Jetpack Compose via Expo Modules
- **expo-ui-swift-ui** - Build native iOS UI with SwiftUI via Expo Modules
- **native-data-fetching** - Efficient data fetching patterns for React Native
- **use-dom** - Use the `use dom` directive for web components in React Native
- **agent-device** - Connect your coding agent to a physical or virtual device for testing

### Meta

- **writing-skills** - Create new skills following best practices
- **using-react-native-space** - Introduction to the skills system

## The Basic Workflow

1. **brainstorming** - Activates before writing code. Refines rough ideas through questions, explores alternatives, presents design in sections for validation. Saves design document.

2. **using-git-worktrees** - Activates after design approval. Creates isolated workspace on new branch, runs project setup, verifies clean test baseline.

3. **writing-plans** - Activates with approved design. Breaks work into bite-sized tasks (2-5 minutes each). Every task has exact file paths, complete code, verification steps.

4. **subagent-driven-development** or **executing-plans** - Activates with plan. Dispatches fresh subagent per task with two-stage review (spec compliance, then code quality), or executes in batches with human checkpoints.

5. **test-driven-development** - Activates during implementation. Enforces RED-GREEN-REFACTOR with React Native Testing Library: write failing test, watch it fail, write minimal code, watch it pass, commit. Deletes code written before tests.

6. **requesting-code-review** - Activates between tasks. Reviews against plan, reports issues by severity. Critical issues block progress.

7. **finishing-a-development-branch** - Activates when tasks complete. Verifies tests, presents options (merge/PR/keep/discard), cleans up worktree.

**The agent checks for relevant skills before any task.** Mandatory workflows, not suggestions.

## Philosophy

- **Test-Driven Development** - Write tests first using React Native Testing Library. Every component, hook, and screen gets tested before implementation.
- **Expo best practices** - Leverage Expo SDK, Expo Router, EAS Build, and the managed workflow wherever possible. Use config plugins and dev clients when native code is needed.
- **Native-first development** - When building custom UI, prefer platform-native approaches (SwiftUI on iOS, Jetpack Compose on Android) via Expo Modules over JavaScript-only workarounds.
- **Device testing with agent-device** - Real device validation is part of the workflow, not an afterthought. The agent-device skill connects your coding agent directly to physical or virtual devices.
- **Systematic over ad-hoc** - Process over guessing. Follow the workflow.
- **Complexity reduction** - Simplicity as primary goal. YAGNI and DRY.
- **Evidence over claims** - Verify before declaring success. Tests pass, the app runs, the device shows the right screen.

## Contributing

Skills live directly in this repository. To contribute:

1. Fork the repository
2. Create a branch for your skill
3. Follow the `writing-skills` skill for creating and testing new skills
4. Submit a PR

See `skills/writing-skills/SKILL.md` for the complete guide.

## Updating

Skills update automatically when you update the plugin:

```bash
/plugin update react-native-space
```

## License

MIT License - see LICENSE file for details
