# Claude Skill: General Code Style Guide

## Purpose
This skill provides coding standards, conventions, and principles for Spring Boot application development. Use this as reference when writing code, reviewing code, or making style decisions.

## General Principles

**Clarity Over Cleverness**
- Write code that's easy to understand, not code that shows off
- If it needs extensive comments to explain, it probably needs refactoring
- Choose obvious solutions over clever ones

**Naming Matters**
- Use descriptive names that reveal intent
- Functions and methods should be verbs: `calculateTotal()`, `validateInput()`
- Classes and types should be nouns: `UserAccount`, `OrderProcessor`
- Booleans should read like questions: `isValid`, `hasPermission`, `canEdit`

**Keep It Small**
- Functions should do one thing well
- Classes should have a single, clear responsibility
- Files should be cohesive and focused

## Code Organization

**File Structure**
- Group related functionality together
- Keep public interfaces at the top
- Internal/private implementation below
- One primary class/concept per file

**Imports/Dependencies**
- Standard library first
- Third-party libraries next
- Local project imports last
- Alphabetize within each group

## Formatting

**Indentation**
- Use consistent indentation (typically 2 or 4 spaces, or tabs—pick one)
- Be consistent across the entire project

**Line Length**
- Aim for 80-120 characters per line
- Break long lines at logical points
- Don't sacrifice readability for line length rules

**Whitespace**
- Use blank lines to separate logical sections
- Add space around operators: `x = a + b` not `x=a+b`
- One statement per line

## Comments and Documentation

**When to Comment**
- Explain *why*, not *what*
- Document non-obvious decisions
- Warn about gotchas or edge cases
- Provide examples for complex APIs

**When NOT to Comment**
- Don't explain what the code obviously does
- Don't leave commented-out code (use version control instead)
- Don't apologize for code quality (fix it instead)

**Documentation**
- Public APIs and interfaces need clear documentation
- Include parameter descriptions and return values
- Provide usage examples for complex functionality

## Error Handling

- Fail fast and loudly during development
- Provide helpful error messages
- Don't swallow exceptions silently
- Validate input at boundaries

## Testing Considerations

- Write testable code
- Avoid tight coupling to external dependencies
- Make dependencies explicit and injectable
- Keep side effects isolated and obvious

## How We Work Best

**When Requirements Are Unclear**
- Ask specific, clarifying questions before coding
- Present tradeoffs with reasoning: "Approach A is simpler; B is more flexible. I recommend A because..."
- Push back when something feels wrong: "This seems too complex—should we refactor first?"

**When Building**
- Start with the domain logic—pure business rules, no framework dependencies
- Keep coordination logic separate from business rules
- Make each component's purpose clear and focused
- Think in conversations—each piece has a distinct role

**When Testing**
- Write tests first
- Test immediately after changes—don't assume it works
- Verify the whole system still functions after changes

**When Things Go Wrong**
- Compare with the last working version first
- Say "I don't know" rather than guess
- Understand why before fixing what

**When Moving Too Fast**
- Pause to verify and test
- Speed without verification leads to long debugging sessions

## Decision Guide

| Question | Act When... |
|----------|-------------|
| Refactor? | Code resists understanding or changes ripple widely |
| Test? | Valuable behaviors need protection or bugs were just fixed |
| Commit? | Tests pass and one logical change is complete |
| Ask? | Requirements unclear, multiple valid paths, or stuck >30 minutes |

## Our Shared Values

**When we disagree** → curiosity leads
**When we're stuck** → we pair debug
**When we succeed** → we learn and update our knowledge base

**Core Principles:**
- Code is a conversation with the future
- Write as if teaching tomorrow's reader
- Tests are living documentation
- Refactoring is an act of care
- Clarity is kindness

## Guiding Foundations

- **SOLID principles** - Well-designed, maintainable code
- **Clean Architecture** - Clear separation of concerns
- **Test-Supported Development** - Confidence through verification
- **Iterative Refinement** - Improve continuously

Clarity before cleverness. Understanding before speed. Behavior before implementation.

