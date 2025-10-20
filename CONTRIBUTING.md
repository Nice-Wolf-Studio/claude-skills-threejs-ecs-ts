# Contributing to Claude Skills: Three.js, ECS, and TypeScript

Thank you for your interest in contributing! This guide will help you create high-quality skills for the repository.

## Skill Structure

Each skill should follow this structure:

```markdown
---
name: skill-name
description: Brief one-line description of what the skill does
---

# Skill Title

## When to Use
[Clear description of scenarios where this skill applies]

## Core Principles
[3-7 key principles that guide this skill's approach]

## Implementation
[Step-by-step guide with code examples]

## Checklist
[TodoWrite-compatible checklist items]

## Common Pitfalls
[What to avoid]

## Performance Tips (if applicable)
[Optimization advice]

## Related Skills
[Links to related skills]
```

## Writing Guidelines

### 1. Clear Scope
- Focus on ONE specific problem or pattern
- Make the "When to Use" section crystal clear
- If the skill tries to do too much, split it into multiple skills

### 2. Actionable Content
- Provide complete, runnable code examples
- Include TypeScript types for everything
- Show real-world usage patterns

### 3. Checklist Format
- Create TodoWrite-compatible checklists with `- [ ]` format
- Each item should be a specific, testable action
- Order items by execution sequence

### 4. Code Quality
- All code examples must be TypeScript
- Use strict TypeScript settings
- Follow modern ES6+ patterns
- Include type safety examples
- Show proper resource cleanup

### 5. Performance Awareness
- Mention performance implications
- Provide optimization tips
- Show cache-friendly patterns when relevant

## Skill Categories

### Three.js Skills (`skills/threejs/`)
- Scene setup and management
- Rendering optimization
- Lighting and materials
- Animation patterns
- Asset loading
- Post-processing effects

### ECS Skills (`skills/ecs/`)
- Architecture patterns
- Component design
- System implementation
- Performance optimization
- Query patterns
- Integration with Three.js

### TypeScript Skills (`skills/typescript/`)
- Type-safe patterns
- Generic utilities
- Branded types
- Runtime type guards
- Configuration typing
- Performance-focused typing

## Submission Process

1. **Fork the Repository**
   ```bash
   gh repo fork Nice-Wolf-Studio/claude-skills-threejs-ecs-ts --clone
   ```

2. **Create a Branch**
   ```bash
   git checkout -b skill/your-skill-name
   ```

3. **Write Your Skill**
   - Place in appropriate category directory
   - Follow the skill structure template
   - Test with Claude Code

4. **Test Your Skill**
   ```bash
   # In Claude Code:
   # 1. Load your skill
   # 2. Test with: "Use the your-skill-name skill to..."
   # 3. Verify checklist items work with TodoWrite
   ```

5. **Update README**
   - Add your skill to the appropriate section in README.md
   - Keep skills alphabetically sorted within sections

6. **Submit Pull Request**
   ```bash
   git add .
   git commit -m "Add [skill-name] skill for [category]"
   git push origin skill/your-skill-name
   gh pr create
   ```

## PR Guidelines

Your pull request should include:

1. **Title Format**: `Add [skill-name] skill for [category]`
2. **Description** with:
   - What problem the skill solves
   - Example use cases
   - Testing performed
3. **Updated README.md** with skill listed
4. **Working code examples** that are tested
5. **No other changes** - one skill per PR

## Code Style

### TypeScript
```typescript
// ✓ Good: Strict types, clear names, readonly when possible
export interface Transform {
  readonly position: Vector3;
  readonly rotation: number;
}

// ✗ Bad: Loose types, unclear names
export interface T {
  pos: any;
  rot: number;
}
```

### Comments
```typescript
// ✓ Good: Explain WHY, not WHAT
// Cap at 2 to prevent performance issues on 4K+ displays
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

// ✗ Bad: Obvious comment
// Set pixel ratio
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
```

### Examples
```typescript
// ✓ Good: Complete, runnable example
const world = new World();
const entity = world.createEntity();
world.addComponent(entity, Transform, new Transform(0, 0, 0));

// ✗ Bad: Incomplete snippet
world.addComponent(entity, transform);
```

## Testing Checklist

Before submitting, verify:

- [ ] Skill follows the standard structure
- [ ] All code examples are complete and runnable
- [ ] TypeScript types are strict and correct
- [ ] Checklist items are TodoWrite-compatible
- [ ] Tested in Claude Code successfully
- [ ] README.md updated with skill listing
- [ ] No typos or grammar errors
- [ ] Performance tips included (if relevant)
- [ ] Related skills linked appropriately

## Questions?

- Open an issue for discussion
- Check existing skills for examples
- Review Claude Code skill documentation

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
