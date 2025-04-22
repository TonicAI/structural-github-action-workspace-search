# Workspace Search

This action searches for workspaces using the Tonic API and returns workspace IDs and names.

## Inputs

- `api_key` (required): Tonic API key for authentication
- `api_url` (optional): Tonic API base URL, defaults to 'https://app.tonic.ai'
- `search_term` (optional): Term to search for in workspace names
- `database_types` (optional): Comma-separated list of database types to filter by (e.g., "Postgres,MySql")
- `tags` (optional): Comma-separated list of tags to filter by (e.g., "production,test")
- `owner_id` (optional): Filter workspaces by owner ID

## Outputs

- `workspaces_json`: JSON string containing workspace IDs and names in GitHub Actions matrix-ready format: `{"workspaces": [{"id":"...","name":"..."},...]}`
- `workspaces_count`: Number of workspaces found

## Example Usage

### Basic Usage

```yaml
jobs:
  search-workspaces:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Search Workspaces
        id: search
        uses: TonicAI/structural-workspace-search@v1
        with:
          api_key: ${{ secrets.TONIC_API_KEY }}

      - name: Print Workspace Count
        run: echo "Found ${{ steps.search.outputs.workspaces_count }} workspaces"
```

### Using the Search Results

```yaml
jobs:
  search-and-process:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Search Workspaces
        id: search
        uses: TonicAI/structural-workspace-search@v1
        with:
          api_key: ${{ secrets.TONIC_API_KEY }}
          search_term: "production"
          database_types: "Postgres,MySQL"
          tags: "prod,critical"
          owner_id: "00000000-0000-0000-0000-000000000000"

      - name: Process Search Results
        run: |
          # Parse the JSON string to a variable
          WORKSPACES='${{ steps.search.outputs.workspaces_json }}'

          # Process the workspaces using jq
          echo "$WORKSPACES" | jq -c '.workspaces[]' | while read -r workspace; do
            id=$(echo "$workspace" | jq -r '.id')
            name=$(echo "$workspace" | jq -r '.name')
            echo "Processing workspace: $name ($id)"

            # Do something with each workspace...
          done
```

### Using Results with GitHub Actions Matrix Strategy

You can use the search results to dynamically create a matrix of workspaces for parallel job execution:

```yaml
jobs:
  # First job to search for workspaces
  search-workspaces:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.create-matrix.outputs.matrix }}
      workspace_count: ${{ steps.search.outputs.workspaces_count }}
    steps:
      - name: Search Workspaces
        id: search
        uses: TonicAI/structural-workspace-search@v1
        with:
          api_key: ${{ secrets.TONIC_API_KEY }}
          tags: "prod" # Only search for production workspaces

      - name: Set Matrix Output
        id: create-matrix
        run: |
          echo "matrix<<EOF" >> $GITHUB_OUTPUT
          echo '${{ steps.search.outputs.workspaces_json }}' >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
          echo "Found ${{ steps.search.outputs.workspaces_count }} workspaces for matrix"

  # Second job that uses the matrix to run tasks in parallel
  process-workspaces:
    needs: search-workspaces
    if: needs.search-workspaces.outputs.workspace_count > 0
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJson(needs.search-workspaces.outputs.matrix) }}
      fail-fast: false # Continue with other workspaces if one fails
    name: Process ${{ matrix.workspaces.name }}
    steps:
      - name: Start Generation Job
        id: start-job
        uses: TonicAI/structural-start-job@v1
        with:
          workspace_id: ${{ matrix.workspaces.id }}
          api_key: ${{ secrets.TONIC_API_KEY }}
          strict_mode: "RejectOnSchemaActions"

      - name: Print Job Details
        run: |
          echo "Started generation job for workspace: ${{ matrix.workspaces.name }}"
          echo "Job ID: ${{ steps.start-job.outputs.job_id }}"
```

This setup allows you to:
1. Search for a set of workspaces that match specific criteria
2. Automatically create a GitHub Actions matrix from those workspaces
3. Run parallel jobs for each workspace
4. Each parallel job has access to both the workspace ID and name