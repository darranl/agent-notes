# Agent Notes Index

This index helps AI agents quickly locate relevant documentation without loading all files.

## Repository Purpose

This repository contains documentation, guides, and context to assist developers working on WildFly, with particular emphasis on WildFly Elytron development. Content covers development workflows, PR review guidelines, issue triage, and best practices for day-to-day engineering work.

## Quick Navigation

### Core Documentation
- **[README.md](README.md)** - Repository overview and purpose
- **[AGENTS.md](AGENTS.md)** - Agent-specific instructions and guidance

### Feature Development Guides

Located in `feature-development/`:

#### Version Management
- **[management-model-version-bump-guide.md](feature-development/management-model-version-bump-guide.md)**
  - **Purpose**: Guide for bumping WildFly subsystem management model versions
  - **Use When**: Making changes to subsystem management API (attributes, operations, capabilities)
  - **Key Topics**: Version bump process, transformer chains, PR review guidelines, stability levels
  - **Related To**: Schema version bumps (often done together)

- **[schema-version-bump-guide.md](feature-development/schema-version-bump-guide.md)**
  - **Purpose**: Guide for bumping WildFly subsystem XML schema versions
  - **Use When**: Changing XML configuration format, adding/removing elements, promoting stability levels
  - **Key Topics**: Schema versioning, XSD files, parser registration, stability level annotations
  - **Related To**: Management model version bumps (often coordinated)

#### Testing Requirements
- **[subsystem-schema-test-requirements.md](feature-development/subsystem-schema-test-requirements.md)**
  - **Purpose**: Requirements for WildFly subsystem schema test files
  - **Use When**: Creating or updating test XML files for schema versions
  - **Key Topics**: Test coverage requirements, element ordering, expression coverage, stability-level testing
  - **Related To**: Schema version bumps (tests required for each schema version)

## Finding Specific Information

### By Task Type

| Task | Primary Guide | Supporting Guides |
|------|--------------|-------------------|
| Bump management model version | management-model-version-bump-guide.md | schema-version-bump-guide.md |
| Bump schema version | schema-version-bump-guide.md | management-model-version-bump-guide.md |
| Create/update test files | subsystem-schema-test-requirements.md | schema-version-bump-guide.md |
| Promote feature stability | Both version bump guides | subsystem-schema-test-requirements.md |
| Review version bump PR | Both version bump guides | - |

### By Keyword

- **Version bump**: management-model-version-bump-guide.md, schema-version-bump-guide.md
- **Transformer**: management-model-version-bump-guide.md
- **Schema/XSD**: schema-version-bump-guide.md
- **Parser**: schema-version-bump-guide.md
- **Test coverage**: subsystem-schema-test-requirements.md
- **Stability levels**: All guides (PREVIEW, COMMUNITY, DEFAULT, EXPERIMENTAL)
- **PR review**: management-model-version-bump-guide.md, schema-version-bump-guide.md
- **WildFly release**: All guides (version targeting)

## Document Structure Conventions

All guides follow these conventions to aid agent navigation:

### Standard Sections
- **Overview** - Purpose and scope
- **Pre-[Action] Checklist** - Prerequisites before starting
- **[Action] Process** - Step-by-step instructions
- **Verification Steps** - How to confirm success
- **Common Patterns** - Best practices and workflows
- **PR Review Guidelines** - What to check when reviewing
- **Troubleshooting** - Common issues and solutions
- **Summary Checklist** - Quick reference for completion

### Heading Markers
- Use `##` for major sections
- Use `###` for subsections
- Use `####` for detailed steps
- Use `**Bold**` for critical requirements
- Use `✅` for correct patterns
- Use `❌` for incorrect patterns

### Code Examples
- Inline code: `ClassName` or `method()`
- Code blocks: Include language identifier (java, xml, bash)
- File paths: Use absolute paths in examples

## Usage Tips for Agents

1. **Start with the index** - Don't load all files immediately
2. **Use keyword search** - Find relevant guide by task or keyword
3. **Check related guides** - Version bumps often require multiple guides
4. **Follow section structure** - Guides use consistent organization
5. **Look for checklists** - Each guide has verification checklists
6. **Check for examples** - Guides include practical examples and patterns

## Maintenance

This index should be updated when:
- New guides are added to the repository
- Existing guides are significantly restructured
- New subdirectories are created
- Guide purposes or scopes change

---

**Last Updated**: 2026-08-21
**Index Version**: 1.0