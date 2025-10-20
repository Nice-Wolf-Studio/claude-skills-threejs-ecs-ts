# Claude Code Skills: Three.js, ECS, and TypeScript

A collection of specialized Claude Code skills for Three.js development, Entity Component Systems (ECS), and TypeScript best practices.

## About

This repository contains skills that extend Claude Code's capabilities for game development and 3D graphics programming with Three.js, implementing ECS architecture patterns, and TypeScript development best practices.

## Available Skills

### Three.js Skills
- `threejs-scene-setup` - Set up a Three.js scene with best practices
- `threejs-optimization` - Performance optimization patterns for Three.js
- `threejs-lighting` - Advanced lighting setup and configuration

### ECS (Entity Component Systems) Skills
- `ecs-architecture` - Design and implement ECS architecture
- `ecs-component-design` - Create efficient component patterns
- `ecs-system-patterns` - Build performant system implementations

### TypeScript Skills
- `typescript-game-types` - Type-safe game development patterns
- `typescript-ecs-types` - Type definitions for ECS architectures
- `typescript-performance` - Performance-focused TypeScript patterns

## Installation

To use these skills with Claude Code:

1. Clone this repository:
   ```bash
   git clone https://github.com/Nice-Wolf-Studio/claude-skills-threejs-ecs-ts.git
   ```

2. Add to your Claude Code plugins directory:
   ```bash
   ln -s /path/to/claude-skills-threejs-ecs-ts ~/.claude/plugins/threejs-ecs-skills
   ```

3. Restart Claude Code or reload the skills

## Usage

Invoke skills in Claude Code using the Skill tool:

```
claude: Use the threejs-scene-setup skill to create a new scene
```

Or directly:
```
/skill threejs-scene-setup
```

## Skill Structure

Each skill follows the Claude Code skill format:

```markdown
---
name: skill-name
description: Brief description of what the skill does
---

# Skill Name

## When to Use
[Description of when to apply this skill]

## Core Principles
[Key concepts and guidelines]

## Implementation
[Step-by-step instructions]

## Examples
[Code examples and use cases]
```

## Contributing

Contributions are welcome! Please:

1. Follow the existing skill structure
2. Include clear examples
3. Test skills with Claude Code
4. Submit a PR with description

## License

MIT License - See LICENSE file for details

## Maintained By

Nice Wolf Studio

## Links

- [Claude Code Documentation](https://docs.claude.com/en/docs/claude-code)
- [Three.js Documentation](https://threejs.org/docs/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
