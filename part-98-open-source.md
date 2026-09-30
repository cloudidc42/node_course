# Part 98 | ขั้นตอนที่ 1761-1780 จาก 1000+

## Open Source Development และ Community Building

ในส่วนนี้เราจะเรียนรู้การ contribute ใน Open Source, การสร้าง OSS projects, และการสร้าง developer community

---

## ขั้นตอนที่ 1761: Creating an Open Source Node.js Library

```javascript
// src/index.js - Entry point for OSS library
// Example: A validation library

'use strict';

const { ValidationError } = require('./errors');
const { validators } = require('./validators');
const { rules } = require('./rules');

/**
 * Create a validator schema
 * @param {Object} schema - Schema definition
 * @returns {Validator} Validator instance
 */
function createValidator(schema) {
  return new Validator(schema);
}

class Validator {
  constructor(schema) {
    this.schema = schema;
    this.customRules = new Map();
  }

  /**
   * Add a custom validation rule
   * @param {string} name - Rule name
   * @param {Function} fn - Validation function
   */
  addRule(name, fn) {
    this.customRules.set(name, fn);
    return this;
  }

  /**
   * Validate data against the schema
   * @param {*} data - Data to validate
   * @throws {ValidationError} When validation fails
   * @returns {Object} Validated and sanitized data
   */
  validate(data) {
    const errors = [];
    const result = {};
    
    for (const [field, rules] of Object.entries(this.schema)) {
      try {
        result[field] = this.validateField(field, data[field], rules);
      } catch (err) {
        errors.push({ field, message: err.message });
      }
    }
    
    if (errors.length > 0) {
      throw new ValidationError('Validation failed', errors);
    }
    
    return result;
  }

  validateField(field, value, rules) {
    let currentValue = value;
    
    for (const [rule, options] of Object.entries(rules)) {
      if (this.customRules.has(rule)) {
        if (!this.customRules.get(rule)(currentValue, options)) {
          throw new Error(`Failed custom rule: ${rule}`);
        }
      } else if (validators[rule]) {
        currentValue = validators[rule](currentValue, options, field);
      } else {
        throw new Error(`Unknown rule: ${rule}`);
      }
    }
    
    return currentValue;
  }
}

module.exports = { createValidator, Validator, ValidationError };
```

---

## ขั้นตอนที่ 1762: Package.json Best Practices

```json
{
  "name": "@myorg/validator",
  "version": "1.0.0",
  "description": "Fast, type-safe validation library for Node.js",
  "main": "dist/index.js",
  "module": "dist/index.esm.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.esm.js",
      "require": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "files": [
    "dist",
    "README.md",
    "CHANGELOG.md",
    "LICENSE"
  ],
  "scripts": {
    "build": "rollup -c",
    "test": "jest --coverage",
    "test:watch": "jest --watch",
    "lint": "eslint src/**/*.js",
    "format": "prettier --write src/**/*.js",
    "docs": "jsdoc -c jsdoc.json",
    "prepublishOnly": "npm run build && npm test",
    "release": "standard-version"
  },
  "keywords": [
    "validation",
    "schema",
    "nodejs",
    "typescript"
  ],
  "author": {
    "name": "Your Name",
    "email": "you@example.com",
    "url": "https://yourwebsite.com"
  },
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/myorg/validator.git"
  },
  "bugs": {
    "url": "https://github.com/myorg/validator/issues"
  },
  "homepage": "https://myorg.github.io/validator",
  "engines": {
    "node": ">=16.0.0"
  },
  "peerDependencies": {
    "typescript": ">=4.0"
  },
  "peerDependenciesMeta": {
    "typescript": {
      "optional": true
    }
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "eslint": "^8.0.0",
    "prettier": "^3.0.0",
    "rollup": "^3.0.0",
    "standard-version": "^9.0.0",
    "typescript": "^5.0.0"
  }
}
```

---

## ขั้นตอนที่ 1763: GitHub Actions สำหรับ OSS Projects

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Test (Node ${{ matrix.node }})
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [18, 20, 22]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Upload coverage
        if: matrix.node == '20'
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  publish:
    name: Publish to npm
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Publish (if version changed)
        run: |
          PUBLISHED_VERSION=$(npm view $(node -p "require('./package.json').name") version 2>/dev/null || echo "0.0.0")
          CURRENT_VERSION=$(node -p "require('./package.json').version")
          if [ "$CURRENT_VERSION" != "$PUBLISHED_VERSION" ]; then
            npm publish --access public
          fi
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## ขั้นตอนที่ 1764-1780: Community Management และ Documentation

```javascript
// scripts/generate-changelog.js
// Auto-generate changelog from commits using Conventional Commits

const { execSync } = require('child_process');

class ChangelogGenerator {
  constructor() {
    this.conventionalTypes = {
      feat: '✨ Features',
      fix: '🐛 Bug Fixes',
      docs: '📚 Documentation',
      style: '💄 Styles',
      refactor: '♻️ Code Refactoring',
      perf: '⚡ Performance Improvements',
      test: '✅ Tests',
      chore: '🔧 Chores',
      ci: '👷 CI/CD',
      breaking: '💥 BREAKING CHANGES'
    };
  }

  getCommitsSinceTag(tag) {
    try {
      const output = execSync(
        `git log ${tag}..HEAD --format="%H|%s|%b" --no-merges`,
        { encoding: 'utf8' }
      );
      
      return output.split('\n')
        .filter(Boolean)
        .map(line => {
          const [hash, subject, ...bodyParts] = line.split('|');
          return { hash: hash.substring(0, 8), subject, body: bodyParts.join('|') };
        });
    } catch {
      return [];
    }
  }

  parseCommit(commit) {
    const match = commit.subject.match(/^(\w+)(?:\((.+)\))?(!)?:\s*(.+)$/);
    if (!match) return null;
    
    return {
      type: match[1],
      scope: match[2] || null,
      breaking: !!match[3] || commit.body?.includes('BREAKING CHANGE'),
      description: match[4],
      hash: commit.hash
    };
  }

  generate(fromTag, toTag = 'HEAD') {
    const commits = this.getCommitsSinceTag(fromTag);
    const parsed = commits.map(c => this.parseCommit(c)).filter(Boolean);
    
    const grouped = {};
    for (const commit of parsed) {
      const key = commit.breaking ? 'breaking' : commit.type;
      if (!grouped[key]) grouped[key] = [];
      grouped[key].push(commit);
    }
    
    const date = new Date().toISOString().split('T')[0];
    const version = require('../package.json').version;
    
    let changelog = `## [${version}] - ${date}\n\n`;
    
    const order = ['breaking', 'feat', 'fix', 'perf', 'docs', 'refactor', 'chore'];
    
    for (const type of order) {
      if (!grouped[type]?.length) continue;
      
      changelog += `### ${this.conventionalTypes[type] || type}\n\n`;
      
      for (const commit of grouped[type]) {
        const scope = commit.scope ? `**${commit.scope}:** ` : '';
        changelog += `- ${scope}${commit.description} (\`${commit.hash}\`)\n`;
      }
      
      changelog += '\n';
    }
    
    return changelog;
  }
}

// Contribution Guide generator
function generateContributingMd(projectName) {
  return `# Contributing to ${projectName}

Thank you for considering contributing! Here's how you can help.

## Development Setup

\`\`\`bash
git clone https://github.com/org/${projectName.toLowerCase()}.git
cd ${projectName.toLowerCase()}
npm install
npm test
\`\`\`

## Development Workflow

1. **Fork** the repository
2. **Create** a feature branch: \`git checkout -b feat/your-feature\`
3. **Make** your changes
4. **Test** thoroughly: \`npm test\`
5. **Commit** using Conventional Commits format
6. **Push** and create a Pull Request

## Commit Message Format

We use [Conventional Commits](https://www.conventionalcommits.org/):

\`\`\`
type(scope): description

feat: add new validation rule
fix(email): handle edge case for international emails
docs: update README examples
\`\`\`

Types: \`feat\`, \`fix\`, \`docs\`, \`style\`, \`refactor\`, \`test\`, \`chore\`

## Code Style

- We use ESLint and Prettier
- Run \`npm run lint\` before committing
- Maintain test coverage above 90%

## Reporting Bugs

Use the [bug report template](.github/ISSUE_TEMPLATE/bug_report.md)

Include:
- Node.js version
- Package version
- Minimal reproduction case

## Feature Requests

Use the [feature request template](.github/ISSUE_TEMPLATE/feature_request.md)

## Questions?

Open a [Discussion](https://github.com/org/${projectName.toLowerCase()}/discussions)
`;
}

module.exports = { ChangelogGenerator, generateContributingMd };
```

```yaml
# .github/ISSUE_TEMPLATE/bug_report.yml
name: Bug Report
description: File a bug report
title: "[Bug]: "
labels: ["bug", "triage"]
body:
  - type: markdown
    attributes:
      value: Thanks for taking the time to fill out this bug report!

  - type: input
    id: version
    attributes:
      label: Package Version
      placeholder: "1.2.3"
    validations:
      required: true

  - type: input
    id: node-version
    attributes:
      label: Node.js Version
      placeholder: "v20.0.0"
    validations:
      required: true

  - type: textarea
    id: description
    attributes:
      label: Describe the bug
      description: A clear and concise description of what the bug is.
    validations:
      required: true

  - type: textarea
    id: reproduction
    attributes:
      label: Minimal Reproduction
      description: Please provide a minimal code snippet to reproduce the issue.
      render: javascript
    validations:
      required: true

  - type: textarea
    id: expected
    attributes:
      label: Expected behavior
    validations:
      required: true
```

---

## แบบฝึกหัด

### Exercise 1: NPM Package
สร้าง และ publish npm package ที่ทำงานจริง (เช่น utility library สำหรับ Node.js)

### Exercise 2: Automated Release
ตั้งค่า semantic-release หรือ changesets สำหรับ automated versioning และ changelog

### Exercise 3: Documentation Site
สร้าง documentation site ด้วย Docusaurus หรือ VitePress สำหรับ OSS project

### คำถามทบทวน
1. Conventional Commits ช่วยอะไรใน OSS projects?
2. วิธีเลือก License ที่เหมาะสมสำหรับ OSS project?
3. Community building สำคัญกับ OSS project อย่างไร?

---

*ต่อไป: Part 99 - Career Path*
