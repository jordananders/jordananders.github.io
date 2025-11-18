---
layout: default
title:  "Building a Code Review Assistant"
date:   2025-11-17 17:00:00
categories: AI LLM Development Tools
---

AI-powered code review assistants can reduce review time by 40% while catching 25% more critical issues. Here's how to build one that integrates with GitHub and provides actionable feedback.

## Architecture Overview

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────┐
│  GitHub PR      │────▶│  GitHub      │────▶│  Code Diff  │
│  Event          │     │  Actions     │     │  Analyzer   │
└─────────────────┘     └──────────────┘     └──────┬──────┘
                                                    │
                                                    ▼
┌─────────────────┐     ┌──────────────┐     ┌─────────────┐
│  PR Comment     │◀────│  Response    │◀────│  LLM API    │
│  Publisher      │     │  Formatter   │     │             │
└─────────────────┘     └──────────────┘     └─────────────┘
```

## GitHub Action Setup

### Basic Workflow

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install openai PyGithub

      - name: Run AI Review
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: python scripts/ai_review.py
```

## Core Implementation

### Diff Analyzer

```python
import subprocess
from dataclasses import dataclass

@dataclass
class FileChange:
    filename: str
    status: str  # added, modified, deleted
    additions: int
    deletions: int
    patch: str

class DiffAnalyzer:
    def __init__(self, base_ref: str, head_ref: str):
        self.base_ref = base_ref
        self.head_ref = head_ref

    def get_changed_files(self) -> list[FileChange]:
        """Get list of changed files with diffs."""
        # Get diff
        result = subprocess.run(
            ["git", "diff", f"{self.base_ref}...{self.head_ref}", "--name-status"],
            capture_output=True, text=True
        )

        files = []
        for line in result.stdout.strip().split("\n"):
            if not line:
                continue

            parts = line.split("\t")
            status = parts[0]
            filename = parts[1] if len(parts) > 1 else ""

            # Get patch for this file
            patch_result = subprocess.run(
                ["git", "diff", f"{self.base_ref}...{self.head_ref}", "--", filename],
                capture_output=True, text=True
            )

            # Count additions/deletions
            additions = sum(1 for l in patch_result.stdout.split("\n") if l.startswith("+") and not l.startswith("+++"))
            deletions = sum(1 for l in patch_result.stdout.split("\n") if l.startswith("-") and not l.startswith("---"))

            files.append(FileChange(
                filename=filename,
                status=self._parse_status(status),
                additions=additions,
                deletions=deletions,
                patch=patch_result.stdout
            ))

        return files

    def _parse_status(self, status: str) -> str:
        status_map = {
            "A": "added",
            "M": "modified",
            "D": "deleted",
            "R": "renamed"
        }
        return status_map.get(status[0], "modified")
```

### LLM Review Engine

```python
from openai import OpenAI
import json

class CodeReviewer:
    def __init__(self, model: str = "gpt-4o"):
        self.client = OpenAI()
        self.model = model

    def review_file(self, file_change: FileChange, context: dict = None) -> dict:
        """Review a single file change."""
        system_prompt = """You are a senior software engineer performing a code review.

        Analyze the code diff and provide:
        1. A brief summary of changes
        2. Potential issues (bugs, security, performance)
        3. Suggestions for improvement
        4. Positive observations

        Be constructive and specific. Reference line numbers when possible.
        Focus on significant issues, not style nitpicks.

        Return JSON with:
        - summary: string
        - issues: [{severity: "critical"|"warning"|"info", line: number|null, message: string}]
        - suggestions: [{line: number|null, message: string}]
        - positives: [string]"""

        user_prompt = f"""Review this code change:

        File: {file_change.filename}
        Status: {file_change.status}

        Diff:
        ```
        {file_change.patch[:8000]}  # Truncate for token limits
        ```

        {"Additional context: " + json.dumps(context) if context else ""}"""

        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt}
            ],
            response_format={"type": "json_object"},
            temperature=0.3
        )

        return json.loads(response.choices[0].message.content)

    def generate_summary(self, file_reviews: list[dict], pr_info: dict) -> str:
        """Generate overall PR summary."""
        prompt = f"""Based on these individual file reviews, create an overall PR summary.

        PR Title: {pr_info.get('title', 'N/A')}
        PR Description: {pr_info.get('body', 'N/A')}

        File Reviews:
        {json.dumps(file_reviews, indent=2)}

        Provide:
        1. Overall assessment (approve/request changes/needs discussion)
        2. Key findings summary
        3. Priority action items

        Keep it concise and actionable."""

        response = self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3
        )

        return response.choices[0].message.content
```

### GitHub Integration

```python
from github import Github
import os

class GitHubReviewPublisher:
    def __init__(self):
        self.gh = Github(os.environ["GITHUB_TOKEN"])
        repo_name = os.environ["GITHUB_REPOSITORY"]
        self.repo = self.gh.get_repo(repo_name)

    def publish_review(
        self,
        pr_number: int,
        summary: str,
        file_reviews: list[dict],
        file_changes: list[FileChange]
    ):
        """Publish review comments to PR."""
        pr = self.repo.get_pull(pr_number)

        # Create main review comment
        body = f"## AI Code Review\n\n{summary}\n\n---\n\n"
        body += "*This review was generated by an AI assistant. Please use your judgment.*"

        # Add inline comments for issues
        comments = []
        for file_change, review in zip(file_changes, file_reviews):
            for issue in review.get("issues", []):
                if issue.get("line"):
                    comments.append({
                        "path": file_change.filename,
                        "line": issue["line"],
                        "body": f"**{issue['severity'].upper()}**: {issue['message']}"
                    })

        # Submit review
        pr.create_review(
            body=body,
            event="COMMENT",
            comments=comments
        )

    def add_file_comment(self, pr_number: int, filename: str, review: dict):
        """Add comment with file-specific review."""
        pr = self.repo.get_pull(pr_number)

        comment = f"### Review: `{filename}`\n\n"
        comment += f"**Summary:** {review['summary']}\n\n"

        if review.get("issues"):
            comment += "**Issues:**\n"
            for issue in review["issues"]:
                emoji = {"critical": "🔴", "warning": "🟡", "info": "🔵"}.get(issue["severity"], "")
                comment += f"- {emoji} {issue['message']}\n"
            comment += "\n"

        if review.get("suggestions"):
            comment += "**Suggestions:**\n"
            for suggestion in review["suggestions"]:
                comment += f"- {suggestion['message']}\n"
            comment += "\n"

        if review.get("positives"):
            comment += "**Positives:** "
            comment += ", ".join(review["positives"])

        pr.create_issue_comment(comment)
```

### Main Script

```python
# scripts/ai_review.py
import os
import json

def main():
    # Get PR info from GitHub event
    event_path = os.environ.get("GITHUB_EVENT_PATH")
    with open(event_path) as f:
        event = json.load(f)

    pr_number = event["pull_request"]["number"]
    base_ref = event["pull_request"]["base"]["sha"]
    head_ref = event["pull_request"]["head"]["sha"]
    pr_info = {
        "title": event["pull_request"]["title"],
        "body": event["pull_request"]["body"]
    }

    # Get diffs
    analyzer = DiffAnalyzer(base_ref, head_ref)
    file_changes = analyzer.get_changed_files()

    # Filter reviewable files
    reviewable_extensions = {".py", ".js", ".ts", ".go", ".java", ".rs"}
    file_changes = [
        f for f in file_changes
        if any(f.filename.endswith(ext) for ext in reviewable_extensions)
    ]

    if not file_changes:
        print("No reviewable files found")
        return

    # Review each file
    reviewer = CodeReviewer()
    file_reviews = []

    for file_change in file_changes:
        print(f"Reviewing {file_change.filename}...")
        review = reviewer.review_file(file_change)
        file_reviews.append(review)

    # Generate summary
    summary = reviewer.generate_summary(file_reviews, pr_info)

    # Publish to GitHub
    publisher = GitHubReviewPublisher()
    publisher.publish_review(pr_number, summary, file_reviews, file_changes)

    print(f"Review published to PR #{pr_number}")

if __name__ == "__main__":
    main()
```

## Advanced Features

### Context-Aware Review

```python
class ContextAwareReviewer(CodeReviewer):
    def __init__(self, model: str = "gpt-4o"):
        super().__init__(model)
        self.codebase_context = {}

    def load_context(self, repo_path: str):
        """Load codebase context for better reviews."""
        # Load README for project understanding
        readme_path = os.path.join(repo_path, "README.md")
        if os.path.exists(readme_path):
            with open(readme_path) as f:
                self.codebase_context["readme"] = f.read()[:2000]

        # Load style guide if exists
        for style_file in [".eslintrc.json", "pyproject.toml", ".editorconfig"]:
            path = os.path.join(repo_path, style_file)
            if os.path.exists(path):
                with open(path) as f:
                    self.codebase_context["style_guide"] = f.read()[:1000]
                break

        # Load recent related files
        self.codebase_context["patterns"] = self._extract_patterns(repo_path)

    def _extract_patterns(self, repo_path: str) -> str:
        """Extract coding patterns from codebase."""
        # Simplified - could use AST analysis
        return "Common patterns extracted from codebase..."

    def review_file(self, file_change: FileChange, context: dict = None) -> dict:
        # Merge codebase context
        full_context = {**self.codebase_context, **(context or {})}
        return super().review_file(file_change, full_context)
```

### Security-Focused Review

```python
SECURITY_PATTERNS = [
    {
        "pattern": r"(password|secret|api_key)\s*=\s*['\"][^'\"]+['\"]",
        "message": "Hardcoded credential detected",
        "severity": "critical"
    },
    {
        "pattern": r"eval\s*\(",
        "message": "Use of eval() is dangerous",
        "severity": "critical"
    },
    {
        "pattern": r"subprocess\.call\(.*, shell=True",
        "message": "Shell injection risk with shell=True",
        "severity": "warning"
    },
    {
        "pattern": r"# TODO|# FIXME|# HACK",
        "message": "Unresolved TODO/FIXME found",
        "severity": "info"
    }
]

class SecurityReviewer:
    def check_patterns(self, patch: str) -> list[dict]:
        """Check for security patterns in code."""
        import re
        issues = []

        for pattern_def in SECURITY_PATTERNS:
            matches = re.finditer(pattern_def["pattern"], patch, re.IGNORECASE)
            for match in matches:
                # Find line number
                line_num = patch[:match.start()].count("\n") + 1
                issues.append({
                    "severity": pattern_def["severity"],
                    "line": line_num,
                    "message": pattern_def["message"]
                })

        return issues
```

### Multi-Model Review

```python
class MultiModelReviewer:
    def __init__(self):
        self.models = {
            "general": "gpt-4o",
            "security": "gpt-4o",
            "performance": "gpt-4o-mini"
        }

    async def comprehensive_review(self, file_change: FileChange) -> dict:
        """Run multiple specialized reviews."""
        import asyncio

        tasks = [
            self._review_aspect(file_change, "general", "Review for code quality and best practices"),
            self._review_aspect(file_change, "security", "Focus on security vulnerabilities"),
            self._review_aspect(file_change, "performance", "Analyze for performance issues")
        ]

        results = await asyncio.gather(*tasks)

        return {
            "general": results[0],
            "security": results[1],
            "performance": results[2]
        }

    async def _review_aspect(self, file_change: FileChange, aspect: str, focus: str) -> dict:
        """Review specific aspect of code."""
        client = OpenAI()

        response = await client.chat.completions.create(
            model=self.models[aspect],
            messages=[
                {"role": "system", "content": f"You are reviewing code. {focus}"},
                {"role": "user", "content": f"Review:\n{file_change.patch}"}
            ],
            response_format={"type": "json_object"}
        )

        return json.loads(response.choices[0].message.content)
```

## Configuration

```yaml
# .github/ai-review-config.yml
enabled: true
model: "gpt-4o"

# File filters
include:
  - "**/*.py"
  - "**/*.js"
  - "**/*.ts"
  - "**/*.go"

exclude:
  - "**/test/**"
  - "**/migrations/**"
  - "**/*.min.js"

# Review settings
max_files: 20
max_file_size: 10000  # characters
focus:
  - security
  - performance
  - best_practices

# Comment settings
inline_comments: true
summary_comment: true
approve_on_pass: false
```

## Limitations & Best Practices

### Limitations

- **No holistic context**: Reviews individual files, may miss architectural issues
- **Token limits**: Large files need truncation
- **Cost**: Each review has API costs

### Best Practices

1. **Narrow scope**: Start with specific checks (style, security)
2. **Human oversight**: AI augments, doesn't replace reviewers
3. **Feedback loop**: Collect feedback to improve prompts
4. **Rate limiting**: Don't overwhelm with comments

## Resources

- [Codedog](https://github.com/codedog-ai/codedog) - Open source code review bot
- [LlamaPReview](https://jetxu-llm.github.io/LlamaPReview-site/) - Context-aware review
- [GitHub Actions API](https://docs.github.com/en/actions)

---

*Questions about building a code review assistant? [Let me know](mailto:jordan@jordananderson.us).*
