# Claude Code Skills: Three.js, ECS, and TypeScript

A comprehensive collection of Claude Code skills for building mobile-optimized Three.js games with Entity Component Systems (ECS), React Three Fiber, and TypeScript.

## About

This repository contains specialized skills that extend Claude Code's capabilities for:
- **Three.js Development** - Scene setup, rendering, materials, textures, and optimization
- **Entity Component Systems** - Data-oriented game architecture
- **React Three Fiber** - Declarative 3D with React
- **Mobile Optimization** - Performance, touch input, adaptive quality
- **TypeScript** - Type-safe game development

Perfect for building high-performance mobile 3D games with modern web technologies.

## Available Skills (14)

### Three.js Skills

#### `threejs-scene-setup`
Complete Three.js scene initialization with best practices:
- Renderer, camera, lighting configuration
- Resource management and disposal
- Window resize handling
- Animation loop with start/stop controls
- Performance optimizations for mobile

#### `threejs-geometry-management`
Efficient geometry creation and management:
- BufferGeometry best practices
- InstancedMesh for repeated objects
- Procedural geometry generation
- LOD (Level of Detail) systems
- Geometry pooling and caching
- Mobile optimization techniques

#### `threejs-material-systems`
Comprehensive material management:
- All Three.js material types
- Custom shader materials
- Material pooling and reuse
- Mobile-optimized materials
- PBR workflows
- Material instance management

#### `threejs-texture-management`
Texture loading and optimization:
- Texture loading with caching
- KTX2/Basis compression
- Texture atlasing
- Power-of-two optimization
- Mobile texture limits
- Texture streaming

#### `threejs-raycasting`
Mouse and touch interaction:
- Object picking with raycasting
- Click and hover handlers
- Touch event normalization
- Drag and drop implementation
- Layer-based filtering
- Performance optimization

### Mobile Skills

#### `mobile-performance`
Mobile optimization strategies:
- Device capability detection
- Quality preset system (low/medium/high)
- Adaptive quality scaling
- Performance monitoring
- Battery-aware optimization
- Thermal throttling handling

#### `touch-input-handling`
Mobile touch controls:
- Touch event management
- Gesture detection (swipe, pinch, rotate)
- Virtual joystick implementation
- Multi-touch support
- Touch-to-action mapping
- Mouse fallback for desktop

### React Three Fiber Skills

#### `react-three-fiber-setup`
R3F project setup and configuration:
- TypeScript configuration
- Mobile-optimized Canvas setup
- Adaptive DPR (device pixel ratio)
- Quality presets by device tier
- Custom hooks for game logic
- Reusable component patterns

#### `r3f-ecs-integration`
Integrating ECS with React Three Fiber:
- ECS React context provider
- Entity rendering from ECS data
- ECS game loop in R3F
- Event system for React updates
- Component sync strategies
- Performance optimization

### ECS Skills

#### `ecs-architecture`
Complete ECS implementation:
- EntityManager for entity lifecycle
- ComponentManager with type safety
- System base class with queries
- World orchestrator
- Example components and systems
- Integration with Three.js

#### `ecs-component-patterns`
Advanced component design:
- Tag components (flags)
- Singleton components (global state)
- Composite components
- Temporal components (Lifetime, Cooldown)
- Component pooling
- Relationship components (Parent/Child)

### TypeScript Skills

#### `typescript-game-types`
Type-safe game development:
- Branded types for IDs
- Discriminated unions for states
- Type-safe event systems
- Component type safety
- Immutable vector types
- Runtime type guards

### Game Systems

#### `input-system`
Unified input handling:
- Keyboard, mouse, touch, gamepad support
- Action mapping system
- Input buffering for combos
- Frame-perfect input sampling
- Multiple device abstraction
- Player input component

## Installation

### Method 1: Clone Repository

```bash
# Clone the repository
git clone https://github.com/Nice-Wolf-Studio/claude-skills-threejs-ecs-ts.git

# Link to Claude Code plugins directory
ln -s $(pwd)/claude-skills-threejs-ecs-ts ~/.claude/plugins/threejs-ecs-skills
```

### Method 2: Direct Download

```bash
# Download as ZIP
curl -L https://github.com/Nice-Wolf-Studio/claude-skills-threejs-ecs-ts/archive/refs/heads/main.zip -o skills.zip

# Extract and link
unzip skills.zip
ln -s $(pwd)/claude-skills-threejs-ecs-ts-main ~/.claude/plugins/threejs-ecs-skills
```

## Usage

Invoke skills in Claude Code using the Skill tool:

```
You: Use the threejs-scene-setup skill to create a new scene
```

Or reference directly when describing your task:

```
You: Set up a mobile-optimized Three.js game with ECS architecture
```

Claude Code will automatically use relevant skills based on your request.

## Quick Start: Mobile ECS Game

```typescript
// 1. Install dependencies
npm create vite@latest my-game -- --template react-ts
cd my-game
npm install three @react-three/fiber @react-three/drei

// 2. Use mobile-performance skill for device detection
// 3. Use react-three-fiber-setup skill for Canvas setup
// 4. Use ecs-architecture skill for game state
// 5. Use r3f-ecs-integration skill to connect them
// 6. Use input-system skill for controls
// 7. Use touch-input-handling skill for mobile input
```

Full examples are included in each skill's documentation.

## Skill Structure

Each skill follows a consistent format:

```markdown
---
name: skill-name
description: Brief description
---

# Skill Title

## When to Use
[When to apply this skill]

## Core Principles
[Key concepts]

## Implementation
[Step-by-step code examples]

## Checklist
- [ ] TodoWrite-compatible items

## Common Pitfalls
[What to avoid]

## Performance Tips
[Optimization advice]

## Related Skills
[Links to related skills]
```

## Roadmap

See [SKILLS_ROADMAP.md](./SKILLS_ROADMAP.md) for planned skills.

### Coming Soon
- `collision-system` - Spatial partitioning, broadphase
- `physics-system` - Rapier/Cannon integration
- `camera-system` - Follow camera, cinematics
- `spawn-system` - Object pooling, procedural generation
- `threejs-animation-systems` - Character animation
- `threejs-model-loading` - GLTF, FBX loading
- And 40+ more skills!

## Contributing

Contributions are welcome! See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

### Contribution Workflow

1. Fork the repository
2. Create a feature branch (`git checkout -b skill/new-skill`)
3. Write your skill following the standard structure
4. Test with Claude Code
5. Update README.md
6. Submit a pull request

## Example Projects

### Mobile ECS Game Template

```typescript
// Uses these skills:
// - react-three-fiber-setup
// - mobile-performance
// - ecs-architecture
// - r3f-ecs-integration
// - input-system
// - touch-input-handling

import { Game } from './Game';

function App() {
  return <Game />;
}
```

See skills for complete implementation examples.

## Requirements

- Node.js 18+
- TypeScript 5+
- Three.js r160+
- React 18+
- @react-three/fiber 8+

## Browser Support

- Chrome/Edge 90+
- Safari 15+ (iOS 15+)
- Firefox 90+

Optimized for mobile browsers.

## Performance Targets

- **Desktop**: 60 FPS at 1080p
- **Mobile (High-end)**: 60 FPS at native resolution (capped at 2x DPR)
- **Mobile (Mid-range)**: 60 FPS at 1x DPR
- **Mobile (Low-end)**: 30 FPS with reduced quality

## License

MIT License - See [LICENSE](./LICENSE) file for details.

## Maintained By

[Nice Wolf Studio](https://github.com/Nice-Wolf-Studio)

## Links

- [Claude Code Documentation](https://docs.claude.com/en/docs/claude-code)
- [Three.js Documentation](https://threejs.org/docs/)
- [React Three Fiber](https://docs.pmnd.rs/react-three-fiber)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)

## Support

- **Issues**: [GitHub Issues](https://github.com/Nice-Wolf-Studio/claude-skills-threejs-ecs-ts/issues)
- **Discussions**: [GitHub Discussions](https://github.com/Nice-Wolf-Studio/claude-skills-threejs-ecs-ts/discussions)

## Acknowledgments

Built for the Claude Code community to accelerate Three.js game development with modern patterns and best practices.

---

**Star this repo** if you find it useful! ⭐
