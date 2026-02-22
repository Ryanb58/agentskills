---
name: gh-issues-optimizer
description: GitHub Issues optimization using gh CLI - analyze, organize, and improve issue management
version: 1.0.0
author: agentskills
tags: [github, issues, project-management, cli, organization]
---

# GitHub Issues Optimizer Skill

Analyze and optimize GitHub Issues using the `gh` CLI. Get insights on issue health, stale items, and organization recommendations.

## Prerequisites

```bash
# Install GitHub CLI
brew install gh                    # macOS
apt-get install gh                 # Ubuntu/Debian
choco install gh                   # Windows

# Authenticate
gh auth login
gh auth status                     # Verify authentication
```

## Quick Issue Overview

### Repository Health Check

```bash
# Basic issue counts by state
gh issue list --repo owner/repo --state all --limit 1000 --json number,state,createdAt,updatedAt,labels | jq -r '
  group_by(.state) | 
  map({state: .[0].state, count: length}) | 
  .[] | "\(.state): \(.count)"
'

# Quick dashboard
repo_stats() {
  local repo=$1
  echo "=== Issue Dashboard for $repo ==="
  
  open=$(gh issue list --repo $repo --state open --limit 1000 | wc -l)
  closed=$(gh issue list --repo $repo --state closed --limit 1000 | wc -l)
  
  echo "Open issues: $open"
  echo "Closed issues: $closed"
  echo "Total: $((open + closed))"
  echo "Close rate: $(echo "scale=1; $closed * 100 / ($open + $closed)" | bc)%"
}

repo_stats "owner/repo"
```

### Issue Age Analysis

```bash
# Find stale issues (no activity >30 days)
gh issue list --repo owner/repo --state open --search "updated:<$(date -v-30d +%Y-%m-%d)" --limit 100

# Issues by age buckets
analyze_issue_age() {
  local repo=$1
  local days=$2
  
  echo "Issues inactive for >$days days:"
  gh issue list --repo $repo --state open \
    --search "updated:<$(date -v-${days}d +%Y-%m-%d)" \
    --json number,title,updatedAt,labels,assignees \
    --template '{{range .}}#{{.number}}: {{.title}} (last updated: {{.updatedAt}}){{"\n"}}{{end}}'
}

analyze_issue_age "owner/repo" 30
```

## Issue Organization

### Label Analysis

```bash
# List all labels with usage count
gh label list --repo owner/repo --json name,color,description | jq -r '.[] | "\(.name): \(.description)"'

# Issues without labels (need triage)
gh issue list --repo owner/repo --state open --label "!*" --limit 100

# Issues by label priority
gh issue list --repo owner/repo --state open --label "priority/high" --limit 50
gh issue list --repo owner/repo --state open --label "priority/medium" --limit 50
gh issue list --repo owner/repo --state open --label "priority/low" --limit 50

# Bulk label analysis
label_analysis() {
  local repo=$1
  
  echo "=== Label Distribution ==="
  gh issue list --repo $repo --state open --limit 1000 --json labels | jq -r '
    [.[] | .labels[].name] | 
    group_by(.) | 
    map({label: .[0], count: length}) | 
    sort_by(-.count) | 
    .[] | "\(.count)\t\(.label)"
  '
}

label_analysis "owner/repo"
```

### Milestone Tracking

```bash
# List milestones with progress
gh api repos/owner/repo/milestones --jq '.[] | "\(.title): \(.open_issues) open, \(.closed_issues) closed, \(.state)"'

# Issues without milestone (backlog)
gh issue list --repo owner/repo --state open --milestone "!*" --limit 100

# Overdue milestones
gh api repos/owner/repo/milestones --jq '.[] | select(.due_on != null and .state == "open") | "\(.title) - Due: \(.due_on) - Open: \(.open_issues)"'

# Milestone completion status
milestone_status() {
  local repo=$1
  
  echo "=== Milestone Progress ==="
  gh api repos/$repo/milestones --jq '.[] | select(.state == "open") | {
    title: .title,
    total: (.open_issues + .closed_issues),
    open: .open_issues,
    closed: .closed_issues,
    percent: (if (.open_issues + .closed_issues) > 0 then (.closed_issues * 100 / (.open_issues + .closed_issues)) else 0 end),
    due: .due_on
  } | "\(.title): \(.closed)/\(.total) (\(.percent | floor)%) - Due: \(.due // "No date")"'
}

milestone_status "owner/repo"
```

### Assignee Workload

```bash
# Issues by assignee
assignee_workload() {
  local repo=$1
  
  echo "=== Open Issues by Assignee ==="
  gh issue list --repo $repo --state open --limit 1000 --json assignees | jq -r '
    [.[] | .assignees[].login // "unassigned"] | 
    group_by(.) | 
    map({assignee: .[0], count: length}) | 
    sort_by(-.count) | 
    .[] | "\(.count)\t\(.assignee)"
  '
}

assignee_workload "owner/repo"

# Find overloaded team members (more than 10 open issues)
find_overloaded() {
  local repo=$1
  local threshold=${2:-10}
  
  gh issue list --repo $repo --state open --limit 1000 --json assignees | jq -r \
    --arg threshold "$threshold" '
    [.[] | .assignees[].login // empty] | 
    group_by(.) | 
    map({assignee: .[0], count: length}) | 
    map(select(.count >= ($threshold | tonumber))) | 
    sort_by(-.count) | 
    .[] | "⚠️  \(.assignee) has \(.count) open issues (threshold: \($threshold))"
  '
}

find_overloaded "owner/repo" 10
```

## Issue Quality Metrics

### Description Quality

```bash
# Issues with short descriptions (may need more detail)
gh issue list --repo owner/repo --state open --limit 100 --json number,title,body | jq -r '
  .[] | select((.body // "" | length) < 50) | 
  "#\(.number): \(.title) - Description too short (\(.body // "" | length) chars)"
'

# Issues without templates (missing sections)
gh issue list --repo owner/repo --state open --limit 100 --json number,title,body | jq -r '
  .[] | select((.body // "") | test("(?i)description|steps to reproduce|expected behavior"; "i") | not) | 
  "#\(.number): \(.title) - Missing standard sections"
'
```

### Comment Activity

```bash
# Issues with no comments (may need attention)
gh issue list --repo owner/repo --state open --limit 100 --json number,title,comments | jq -r '.[] | select(.comments == 0) | "#\(.number): \(.title)"'

# Highly discussed issues (hot topics)
gh issue list --repo owner/repo --state open --limit 50 --json number,title,comments | jq -r 'sort_by(-.comments) | .[0:10] | .[] | "\(.comments) comments - #\(.number): \(.title)"'
```

## Optimization Recommendations

### Stale Issue Detection

```bash
# Comprehensive stale issue report
stale_report() {
  local repo=$1
  local days=${2:-30}
  
  echo "=== Stale Issues Report (>$days days inactive) ==="
  echo "Generated: $(date)"
  echo ""
  
  # Stale bugs
  echo "🐛 Stale Bugs:"
  gh issue list --repo $repo --state open --label "bug" \
    --search "updated:<$(date -v-${days}d +%Y-%m-%d)" \
    --json number,title,updatedAt --template '{{range .}}  - #{{.number}}: {{.title}} ({{.updatedAt}}){{"\n"}}{{end}}'
  
  echo ""
  echo "⭐ Stale Feature Requests:"
  gh issue list --repo $repo --state open --label "enhancement" \
    --search "updated:<$(date -v-${days}d +%Y-%m-%d)" \
    --json number,title,updatedAt --template '{{range .}}  - #{{.number}}: {{.title}} ({{.updatedAt}}){{"\n"}}{{end}}'
  
  echo ""
  echo "📋 Stale Untriaged (no labels):"
  gh issue list --repo $repo --state open --label "!*" \
    --search "updated:<$(date -v-${days}d +%Y-%m-%d)" \
    --json number,title,updatedAt --template '{{range .}}  - #{{.number}}: {{.title}} ({{.updatedAt}}){{"\n"}}{{end}}'
}

stale_report "owner/repo" 30
```

### Issue Triage Workflow

```bash
# Untriaged issues needing attention
triage_queue() {
  local repo=$1
  
  echo "=== Triage Queue ==="
  echo ""
  
  echo "🆕 Issues without any labels (need triage):"
  gh issue list --repo $repo --state open --label "!*" --limit 20 \
    --json number,title,createdAt --template '{{range .}}  - #{{.number}}: {{.title}} ({{.createdAt}}){{"\n"}}{{end}}'
  
  echo ""
  echo "🙋 Unassigned issues:"
  gh issue list --repo $repo --state open --assignee "!*" --limit 20 \
    --json number,title,labels --template '{{range .}}  - #{{.number}}: {{.title}} [Labels: {{range .labels}}{{.name}} {{end}}]{{"\n"}}{{end}}'
  
  echo ""
  echo "📅 Issues without milestone:"
  gh issue list --repo $repo --state open --milestone "!*" --limit 20 \
    --json number,title --template '{{range .}}  - #{{.number}}: {{.title}}{{"\n"}}{{end}}'
}

triage_queue "owner/repo"
```

### Duplicate Detection

```bash
# Find potential duplicates by title similarity
find_potential_duplicates() {
  local repo=$1
  
  echo "=== Potential Duplicates (manual review needed) ==="
  gh issue list --repo $repo --state open --limit 100 --json number,title | jq -r '
    .[] | .title |= ascii_downcase |
    group_by(.title) |
    map(select(length > 1)) |
    .[] |
    "Possible duplicates:" + "\n" +
    (map("  - #\(.number): \(.title)") | join("\n"))
  '
}

find_potential_duplicates "owner/repo"
```

## Project Board Integration

```bash
# List projects in repository
gh project list --repo owner/repo

# Issues not on any project board (orphaned work)
orphaned_issues() {
  local repo=$1
  
  echo "=== Issues Not on Project Boards ==="
  # Get all open issues
  gh issue list --repo $repo --state open --limit 200 --json number,title,projectItems | jq -r '
    .[] | select((.projectItems | length) == 0) |
    "#\(.number): \(.title)"
  '
}

orphaned_issues "owner/repo"
```

## Bulk Operations

### Batch Labeling

```bash
# Add label to multiple issues
add_label_batch() {
  local repo=$1
  local label=$2
  shift 2
  
  for issue in "$@"; do
    echo "Adding label '$label' to issue #$issue..."
    gh issue edit $issue --repo $repo --add-label "$label"
  done
}

# Usage: add_label_batch "owner/repo" "triage" 123 124 125
```

### Batch Assignment

```bash
# Assign multiple issues to a team member
assign_batch() {
  local repo=$1
  local assignee=$2
  shift 2
  
  for issue in "$@"; do
    echo "Assigning issue #$issue to @$assignee..."
    gh issue edit $issue --repo $repo --add-assignee "$assignee"
  done
}

# Usage: assign_batch "owner/repo" "username" 123 124 125
```

### Close Stale Issues

```bash
# Close old issues with comment (use with caution!)
close_stale() {
  local repo=$1
  local days=$2
  
  echo "Finding issues stale for >$days days..."
  stale=$(gh issue list --repo $repo --state open \
    --search "updated:<$(date -v-${days}d +%Y-%m-%d)" \
    --json number,title | jq -r '.[] | "\(.number)"')
  
  echo "Found issues: $stale"
  echo "This would close them with a comment. Uncomment to execute."
  
  # for issue in $stale; do
  #   gh issue close $issue --repo $repo --comment "Closing due to inactivity. Please reopen if still relevant."
  # done
}

close_stale "owner/repo" 180
```

## Best Practices

### Recommended Issue Labels

```bash
# Create standard label set
create_standard_labels() {
  local repo=$1
  
  gh label create "priority/high" --repo $repo --color "b60205" --description "High priority - address immediately"
  gh label create "priority/medium" --repo $repo --color "fbca04" --description "Medium priority - address soon"
  gh label create "priority/low" --repo $repo --color "0e8a16" --description "Low priority - address when possible"
  
  gh label create "type/bug" --repo $repo --color "d73a4a" --description "Something isn't working"
  gh label create "type/feature" --repo $repo --color "a2eeef" --description "New feature request"
  gh label create "type/docs" --repo $repo --color "0075ca" --description "Documentation improvement"
  gh label create "type/refactor" --repo $repo --color "c5def5" --description "Code refactoring"
  
  gh label create "status/triage" --repo $repo --color "d876e3" --description "Needs triage"
  gh label create "status/blocked" --repo $repo --color "000000" --description "Blocked by dependencies"
  gh label create "status/wip" --repo $repo --color "ffff00" --description "Work in progress"
  
  gh label create "good-first-issue" --repo $repo --color "7057ff" --description "Good for newcomers"
  gh label create "help-wanted" --repo $repo --color "008672" --description "Extra attention needed"
}

create_standard_labels "owner/repo"
```

### Weekly Health Report

```bash
#!/bin/bash
# weekly_issue_report.sh - Comprehensive issue health report

REPO="owner/repo"

echo "=== Weekly Issue Health Report ==="
echo "Repository: $REPO"
echo "Generated: $(date)"
echo ""

# 1. Open vs Closed
open_count=$(gh issue list --repo $REPO --state open --limit 1000 | wc -l)
closed_count=$(gh issue list --repo $REPO --state closed --limit 1000 | wc -l)
total=$((open_count + closed_count))

echo "📊 Overall Stats:"
echo "  Open: $open_count"
echo "  Closed: $closed_count"
echo "  Total: $total"
if [ $total -gt 0 ]; then
  echo "  Close rate: $(echo "scale=1; $closed_count * 100 / $total" | bc)%"
fi
echo ""

# 2. Untriaged
echo "🆕 Needs Triage:"
gh issue list --repo $REPO --state open --label "!*" --limit 5 --json number,title | jq -r '.[] | "  - #\(.number): \(.title)"'
echo ""

# 3. Stale issues (30+ days)
echo "😴 Stale Issues (>30 days inactive):"
stale_count=$(gh issue list --repo $REPO --state open --search "updated:<$(date -v-30d +%Y-%m-%d)" --limit 1000 | wc -l)
echo "  Count: $stale_count"
gh issue list --repo $REPO --state open --search "updated:<$(date -v-30d +%Y-%m-%d)" --limit 5 --json number,title | jq -r '.[] | "  - #\(.number): \(.title)"'
echo ""

# 4. Overloaded assignees
echo "⚠️ Workload Warnings:"
gh issue list --repo $REPO --state open --limit 1000 --json assignees | jq -r \
  '[.[] | .assignees[].login // empty] | group_by(.) | map({assignee: .[0], count: length}) | map(select(.count >= 10)) | sort_by(-.count) | .[] | "  - \(.assignee): \(.count) issues"'
echo ""

# 5. Milestone status
echo "🎯 Milestones:"
gh api repos/$REPO/milestones --jq '.[] | select(.state == "open") | "  - \(.title): \(.closed_issues)/\(.open_issues + .closed_issues) complete"'
echo ""

echo "=== End of Report ==="
```

## Tips

1. **Set up issue templates** - Guide reporters with `.github/ISSUE_TEMPLATE/`
2. **Use consistent labeling** - Standardize on priority/type/status labels
3. **Review stale issues weekly** - Close or update issues inactive >90 days
4. **Balance workload** - Aim for <10 open issues per assignee
5. **Link PRs to issues** - Use "Fixes #123" in PR descriptions
6. **Use milestones** - Group related work into sprints/releases
7. **Celebrate closures** - Share milestone completions with the team

## Resources

- [GitHub CLI Documentation](https://cli.github.com/manual/)
- [GitHub Issues Best Practices](https://docs.github.com/en/issues/tracking-your-work-with-issues)
- [jq Documentation](https://stedolan.github.io/jq/) for JSON processing
