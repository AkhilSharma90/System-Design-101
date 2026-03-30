# Contributing to System Design 101

Thank you for your interest in contributing to System Design 101. This guide will help you add new courses, sections, and topics to the Algorok learning platform.

## Repository Structure

Courses are organized at the root level of the repository:

```
System-Design-101/
├── intro-to-system-design/         # Course directory
│   ├── course.json                 # Course metadata
│   ├── fundamentals/               # Section directory
│   │   ├── section.json            # Section metadata
│   │   ├── 01-scalability.md       # Topic file
│   │   └── 02-reliability.md       # Topic file
│   └── databases/                  # Section directory
│       ├── section.json
│       └── 01-sql-vs-nosql.md
└── advanced-system-design/         # Another course
    ├── course.json
    └── ...
```

## File Formats

### Course Metadata (`course.json`)

Each course directory must contain a `course.json` file:

```json
{
  "slug": "intro-to-system-design",
  "title": "Introduction to System Design",
  "description": "Learn fundamental system design concepts and patterns",
  "difficulty": "Beginner",
  "duration": "6 weeks",
  "category": "Basics",
  "featured": true,
  "premium": false,
  "available": true
}
```

**Required Fields:**
- `slug`: Unique identifier (kebab-case)
- `title`: Display name of the course
- `description`: Brief course overview
- `difficulty`: "Beginner", "Intermediate", or "Advanced"
- `duration`: Estimated completion time
- `category`: Must be one of the available categories (see below)
- `featured`: Boolean for homepage visibility
- `premium`: Boolean for premium access
- `available`: Boolean for course availability

### Section Metadata (`section.json`)

Each section directory must contain a `section.json` file:

```json
{
  "slug": "fundamentals",
  "title": "System Design Fundamentals",
  "orderIndex": 1
}
```

**Required Fields:**
- `slug`: Unique identifier within the course (kebab-case)
- `title`: Display name of the section
- `orderIndex`: Numerical order in the course (starts at 1)

### Topic Content (`*.md` files)

Topic markdown files use YAML frontmatter for metadata:

```markdown
---
slug: scalability-basics
title: Introduction to Scalability
readTime: 15 min
orderIndex: 1
premium: false
---

# Introduction to Scalability

Your markdown content here...

## Key Concepts

- Vertical scaling vs horizontal scaling
- Load balancing strategies
- Caching patterns
```

**Required Frontmatter:**
- `slug`: Unique identifier within the section (kebab-case)
- `title`: Display name of the topic
- `readTime`: Estimated reading time (e.g., "10 min", "20 min")
- `orderIndex`: Numerical order in the section (starts at 1)
- `premium`: Boolean for premium access

## Available Categories

Use one of these categories in your `course.json`:

- `Basics`
- `Intermediate`
- `LLM System Design`
- `AI Agents`
- `Case Studies`

## File Naming Conventions

- **Course directories**: `kebab-case-course-name`
- **Section directories**: `kebab-case-section-name`
- **Topic files**: `##-topic-name.md` (order prefix optional but recommended)
- **All slugs**: Use kebab-case formatting

## Contribution Workflow

1. Fork this repository
2. Create a feature branch (`git checkout -b add-caching-course`)
3. Add your course content following the structure above
4. Commit your changes (`git commit -am 'Add course: Caching Strategies'`)
5. Push to the branch (`git push origin add-caching-course`)
6. Create a Pull Request

## Content Guidelines

### Quality Standards

- **Original Content**: All content should be original or properly attributed
- **Clarity**: Use clear, descriptive titles and explanations
- **Focus**: Keep topic content focused on a single concept
- **Practical**: Include real-world examples and use cases
- **Code Examples**: Provide code snippets where appropriate
- **Diagrams**: Include diagrams for complex concepts (use markdown-compatible formats)

### Writing Style

- Use inclusive and accessible language
- Write in a professional, educational tone
- Break down complex topics into digestible sections
- Use headings, bullet points, and code blocks for readability
- Maintain consistency with existing content style

### Technical Accuracy

- Ensure technical accuracy of all content
- Reference authoritative sources where applicable
- Include trade-offs and considerations for different approaches
- Update content to reflect current best practices

### Content Organization

- Start with fundamental concepts before advanced topics
- Build upon previously introduced concepts
- Use progressive complexity within courses
- Link related topics when appropriate

## Testing Your Content

Before submitting:

1. Validate JSON files using a JSON validator
2. Check YAML frontmatter syntax
3. Preview markdown rendering locally
4. Verify all links work correctly
5. Ensure proper formatting and readability
6. Check for spelling and grammar errors

## Sync Behavior

Understanding how content syncs to the Algorok platform:

- **Manual Sync**: Platform admins trigger sync from the Manage → GitHub tab
- **Slug Matching**: Content is matched by slugs to preserve database IDs
- **Create or Update**: New content is created, existing content is updated
- **Progress Preservation**: User progress is maintained during content updates
- **Content Overwrite**: Syncing replaces existing content with the GitHub version

**Important**: Once content is published, changing slugs will create duplicate entries rather than updating existing ones.

## Questions or Issues?

If you have questions about contributing or encounter issues, please:

1. Check existing issues for similar questions
2. Review the documentation thoroughly
3. Open a new issue with detailed information
4. Tag appropriately for faster response

Thank you for helping improve System Design 101!
