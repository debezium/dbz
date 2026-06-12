# GitHub Issue Viewer

Interactive web-based tool for viewing and managing GitHub issues with project field editing.

## Features

- View issues created or updated in the last N days (configurable)
- Filter by state (Open/Closed/All)
- Edit labels, project fields (Iteration, Stable Iteration, Product Iteration, Status)
- Add comments to issues
- Auto-loads projects and fields

## Quick Start

1. Open `issue-viewer.html` in your browser
2. Enter repository (e.g., `debezium/dbz`)
3. Enter GitHub token with `repo` and `project` write permissions
4. Select project and days lookback
5. Click "Fetch Issues"

## GitHub Token

Create a Personal Access Token with these scopes:
- `repo` (write) - for labels and comments
- `project` (write) - for project fields

Generate at: GitHub Settings → Developer settings → Personal access tokens

## Usage

- Click issue to view details
- Click "✏️ Edit Issue" to modify labels, project fields, or add comments
- Project fields load automatically
- Changes save immediately

That's it!
