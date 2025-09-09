# GitLint Action Configuration (action.yml) Explained

## Overview

The GitLint Action is a GitHub Action that enforces naming conventions and formatting rules for:
- **Commit messages** (subject and body)
- **Branch names**
- **Pull request titles**
- **Author and committer information**

This action runs on pull requests and validates all commits in the PR against configurable rules using regular expressions, length limits, and other constraints.

## Configuration Parameters

### Basic Settings

| Parameter | Description | Default |
|-----------|-------------|---------|
| `github-token` | GitHub token for API access | `${{ github.token }}` |

### Length Validation Parameters

These parameters control the minimum and maximum character lengths for different parts of commit messages:

| Parameter | Description | Default | Notes |
|-----------|-------------|---------|-------|
| `commit-message-body-max-length` | Max characters per line in commit body | `72` | Set to `-1` to disable |
| `commit-message-body-min-length` | Min characters per line in commit body | `-1` (disabled) | Set to `-1` to disable |
| `commit-message-subject-max-length` | Max characters in commit subject | `72` | Set to `-1` to disable |
| `commit-message-subject-min-length` | Min characters in commit subject | `-1` (disabled) | Set to `-1` to disable |

### Boolean Flag Parameters

These are true/false settings that enable or disable specific validations:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `prohibit-blank-lines-cm-body` | Disallow empty lines in commit message body | `false` |
| `prohibit-unknown-commit-authors` | Require commit authors to be known GitHub users | `false` |
| `prohibit-unknown-commit-committers` | Require commit committers to be known GitHub users | `false` |
| `prohibit-unsigned-commits` | Require commits to have valid GPG signatures | `false` |

### Regular Expression Parameters

These parameters use regex patterns to validate various aspects of commits and pull requests:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `re-branch-name` | Pattern for valid branch names | `.*` (any) |
| `re-commit-author-email` | Pattern for commit author email | `.*` (any) |
| `re-commit-author-name` | Pattern for commit author name | [Complex pattern - see below] |
| `re-commit-committer-email` | Pattern for commit committer email | `.*` (any) |
| `re-commit-committer-name` | Pattern for commit committer name | [Complex pattern - see below] |
| `re-commit-message-body` | Pattern for commit message body | `.*` (any) |
| `re-commit-message-split` | Pattern to separate subject from body | `([^\n]*)(?:\n\n(.*))?` |
| `re-commit-message-subject` | Pattern for commit message subject | [Complex pattern - see below] |
| `re-pull-request-title` | Pattern for pull request title | [Complex pattern - see below] |

## Regular Expression Patterns Explained

### 1. Commit Message Split Pattern
```regex
([^\n]*)(?:\n\n(.*))?
```

**What it does:** Separates the commit message into subject and body parts.

**Plain English:**
- `([^\n]*)` - Capture everything up to the first newline (this is the subject)
- `(?:\n\n(.*))?` - Optionally match two newlines followed by everything else (this is the body)

**Examples:**
- ✅ `"Fix bug in login"` → Subject: "Fix bug in login", Body: none
- ✅ `"Fix bug in login\n\nThis resolves the issue with..."` → Subject: "Fix bug in login", Body: "This resolves the issue with..."

### 2. Commit Author Name Pattern
```regex
^[A-Z][a-z]+[A-Za-z]*(-[A-Z][a-z]+[A-Za-z]*)*( ((St\.|[Vv]an|[Dd]e[lr]?|[Ll]a)|[A-Z](\.|[a-z]+[A-Za-z]*)(-[A-Z](\.|[a-z]+[A-Za-z]*))*))*( [A-Z][a-z]+[A-Za-z]*(-[A-Z][a-z]+[A-Za-z]*)*)*$
```

**What it does:** Validates proper formatting of human names with correct capitalization.

**Plain English Rules:**
1. **Must start with a capital letter** followed by lowercase letters
2. **Can have hyphens** connecting name parts (like "Mary-Jane")
3. **Supports name prefixes** like:
   - "St." (Saint)
   - "van", "Van" (Dutch)
   - "de", "del", "der", "De", "Del", "Der" (various origins)
   - "la", "La" (Spanish/French)
4. **Supports middle names/initials** with proper capitalization
5. **Each word must be properly capitalized** (first letter uppercase, rest lowercase)

**Valid Examples:**
- ✅ `John Smith`
- ✅ `Mary-Jane Watson`
- ✅ `Vincent van Gogh`
- ✅ `Jean de la Fontaine`
- ✅ `John F. Kennedy`
- ✅ `Maria del Carmen`

**Invalid Examples:**
- ❌ `john smith` (not capitalized)
- ❌ `JOHN SMITH` (all caps)
- ❌ `John smith` (second name not capitalized)
- ❌ `J Smith` (first name too short)

### 3. Commit Committer Name Pattern
```regex
^(GitHub|[A-Z][a-z]+[A-Za-z]*(-[A-Z][a-z]+[A-Za-z]*)*( ((St\.|[Vv]an|[Dd]e[lr]?|[Ll]a)|[A-Z](\.|[a-z]+[A-Za-z]*)(-[A-Z](\.|[a-z]+[A-Za-z]*))*))*( [A-Z][a-z]+[A-Za-z]*(-[A-Z][a-z]+[A-Za-z]*)*))$
```

**What it does:** Same as author name pattern, but also allows "GitHub" as a special case.

**Plain English:**
- **Either** the exact word "GitHub"
- **Or** follows all the same rules as the author name pattern above

**Valid Examples:**
- ✅ `GitHub` (special case for GitHub's automated commits)
- ✅ All the same examples as author names above

### 4. Commit Message Subject Pattern
```regex
^(Merge branch .*|(([A-Z0-9]{2,10}-[0-9]+)(, [A-Z0-9]{2,10}-[0-9]+)*: )?\b(?!Made)(?![A-Za-z]*(ing|ung|ang|ed|id|[^s]s)\b)[A-Z][a-z]*\b [/\- ._,()A-Za-z0-9]*[)A-Za-z0-9])$
```

**What it does:** Validates commit message subjects with optional ticket references and specific formatting rules, with special handling for merge commits.

**Plain English Rules:**

**Option 1: Merge Branch Messages**
- **Any message starting with "Merge branch"** is automatically allowed
- **No length restrictions** for merge messages
- **No formatting requirements** for merge messages

**Option 2: Regular Commit Messages**
1. **Optional ticket reference(s):**
   - Format: `PROJ-123: ` or `ABC-456, DEF-789: `
   - 2-10 uppercase letters/numbers, dash, then numbers
   - Multiple tickets separated by commas and spaces
   - Must end with colon and space

2. **Main message requirements:**
   - **Must start with a capital letter**
   - **Cannot start with "Made"** (too vague)
   - **Cannot use gerund forms** (-ing, -ung, -ang endings)
   - **Cannot use past tense** (-ed, -id endings)
   - **Cannot use plural forms** (words ending in 's' except single 's')
   - **First word must be lowercase after the capital** (like "Fix", "Add", "Update")
   - **Can contain various characters** (letters, numbers, spaces, punctuation)
   - **Must end with a letter, number, or closing parenthesis**

**Valid Examples:**
- ✅ `Merge branch 'feature/user-authentication' into main`
- ✅ `Merge branch 'bugfix/login-validation'`
- ✅ `Merge branch 'develop' into 'release/v2.1'`
- ✅ `Fix login bug in authentication module`
- ✅ `Add new user registration feature`
- ✅ `PROJ-123: Update database schema for users`
- ✅ `ABC-456, DEF-789: Refactor payment processing`
- ✅ `Remove deprecated API endpoints`

**Invalid Examples:**
- ❌ `fix login bug` (doesn't start with capital)
- ❌ `Made changes to login` (starts with "Made")
- ❌ `Adding new feature` (uses gerund -ing)
- ❌ `Fixed the bug` (uses past tense -ed)
- ❌ `Updates the code` (plural form)
- ❌ `Fix bug.` (ends with period)

### 5. Pull Request Title Pattern
```regex
^([A-Z0-9]{2,10}-[0-9]+)(, [A-Z0-9]{2,10}-[0-9]+)*: \b(?!Made)(?![A-Za-z]*(ing|ung|ang|ed|id|[^s]s)\b)[A-Z][a-z]*\b [/\- ._,()A-Za-z0-9]*[)A-Za-z0-9]$
```

**What it does:** Validates pull request titles with **required** ticket references.

**Plain English:**
- **Same rules as commit message subject** BUT
- **Ticket reference is REQUIRED** (not optional like in commits)
- Must start with one or more ticket references like `PROJ-123: ` or `ABC-456, DEF-789: `

**Valid Examples:**
- ✅ `PROJ-123: Fix login bug in authentication module`
- ✅ `ABC-456, DEF-789: Add new user registration feature`
- ✅ `TICKET-999: Update database schema for users`

**Invalid Examples:**
- ❌ `Fix login bug` (missing required ticket reference)
- ❌ `PROJ-123: Adding new feature` (uses gerund -ing)
- ❌ `ABC-456: made changes` (starts with lowercase, uses "made")

## Usage Examples

### Strict Configuration Example
```yaml
- name: Lint commits, branches, and pull requests
  uses: aschbacd/gitlint-action@v1.0.2
  with:
    # Enforce length limits
    commit-message-subject-max-length: 50
    commit-message-body-max-length: 72
    
    # Require known GitHub users
    prohibit-unknown-commit-authors: true
    prohibit-unknown-commit-committers: true
    
    # Use default regex patterns (shown above)
    # These enforce proper naming and ticket references
```

### Relaxed Configuration Example
```yaml
- name: Lint commits, branches, and pull requests
  uses: aschbacd/gitlint-action@v1.0.2
  with:
    # Only enforce basic length limits
    commit-message-subject-max-length: 100
    
    # Allow any format for names and messages
    re-commit-author-name: ".*"
    re-commit-committer-name: ".*"
    re-commit-message-subject: ".*"
    re-pull-request-title: ".*"
```

## Common Customizations

### Allow Simpler Name Formats
If the default name patterns are too strict, you can use simpler patterns:
```yaml
re-commit-author-name: "^[A-Z][a-z]+ [A-Z][a-z]+$"  # Just "Firstname Lastname"
```

### Require Different Ticket Formats
For different project management tools:
```yaml
# For Jira-style tickets
re-commit-message-subject: "^[A-Z]+-[0-9]+: [A-Z].*"

# For GitHub issues
re-commit-message-subject: "^#[0-9]+: [A-Z].*"
```

### Allow More Flexible Commit Messages
```yaml
# Just require capitalization, no other restrictions
re-commit-message-subject: "^[A-Z].*"
```

This configuration gives you fine-grained control over your team's Git conventions while maintaining consistency across your codebase.
