# GitHub Repository Evaluator & Intelligent Analyzer

A CLI tool that analyzes GitHub repositories using layered techniques: template-driven static checks, objective scoring, LLM-assisted judgments, and a deep intelligent analyzer that crawls the repo to infer purpose, architecture, coherence, and quality.

## Key capabilities

- Objective scoring across code quality, documentation, functionality, innovation, and UX
- Hackathon eligibility checks (dates, team size, originality, demo presence)
- Project-type templates for focused analysis (Next.js, React, Node.js, Fullstack, Mobile, Custom)
- Intelligent, content-aware analysis of repo purpose, architecture, coherence, and quality
- LLM-assisted evaluations and feasibility judgment (via Together AI)
- Exportable JSON reports

## Quick start

```bash
git clone <this-repo>
cd github agent
npm install

# Configure tokens (required)
# PowerShell (Windows)
$env:GITHUB_TOKEN = "<your-github-token>"
$env:TOGETHER_API_KEY = "<your-together-ai-key>"

# macOS/Linux
export GITHUB_TOKEN="<your-github-token>"
export TOGETHER_API_KEY="<your-together-ai-key>"

# Run interactive CLI
npm start

# Or run the simple example
npm run analyze
```

## How it works: analysis layers

The tool combines several complementary layers. Each layer can work independently; together they provide a holistic evaluation.

### 1) Repository ingestion layer

- Files:
  - `evaluateRepo.js` → `fetchRepoData(owner, repo)` pulls repo metadata, contributors, and commit history from GitHub REST API.
  - `intelligentAnalyzer.js` → `fetchRepositoryStructure(owner, repo)` recursively crawls directories and files, fetches contents for important files, and builds an in-memory structure.
- Responsibilities:
  - Traverse directories while skipping noise (e.g., `node_modules`, build artifacts, binaries)
  - Fetch content for analysis with size/type filters
  - Record lists of all files/directories and retain full text for key files (code, JSON, docs, configs)

### 2) Template-driven static analysis layer

- Files:
  - `projectTemplates.json`: Catalog of project types (Next.js, React, Node.js, Fullstack, Mobile, Custom) with expected files, directories, and evaluation criteria.
  - `evaluateRepo.js` → `getComprehensiveCodeAnalysis(repoData, selectedTemplate)`
- Responsibilities:
  - Auto-detect project type (`detectProjectTypeFromRepo`) or use user-selected template
  - Expand template wildcards (e.g., `pages/api/*`) and fetch content for those files
  - Compute per-file stats (size, line count, type) and a project summary (file count, line count, file-type distribution)

### 3) Objective scoring layer

- Files/Functions (in `evaluateRepo.js`):
  - Code Quality: `calculateCodeQualityScore` → structure, code style, modern features, error handling, type safety
  - Documentation: `calculateDocumentationScore` → README presence/structure, installation, API docs
  - Functionality: `calculateFunctionalityScore` → DB integration, routes, auth, MVC structure
  - Innovation: `calculateInnovationScore` → advanced deps and architecture breadth
  - User Experience: `calculateUserExperienceScore` → docs clarity, usage examples, error handling, CORS
- Weighting:
  - `CRITERIA_WEIGHTS` in `evaluateRepo.js`
  - Final score = weighted sum, normalized to 0–10

### 4) LLM-assisted evaluation layer

- Files/Functions (in `evaluateRepo.js`):
  - Together AI client: `together-ai`
  - Timeboxed calls via `callTogetherAIWithTimeout`
  - AI judgments: `evaluateCodeQualityWithAI`, `evaluateDocumentationWithAI`, `evaluateFunctionalityWithAI`, `evaluateInnovationWithAI`, `evaluateUserExperienceWithAI`
  - Optional Hackathon Feasibility: Prompted summary for “could this be built within the window?”
- Responsibilities:
  - Produce concise expert assessments and an extracted numeric score
  - Fail-safe defaults if timeouts/errors occur

### 5) Intelligent semantic analysis layer

- File: `intelligentAnalyzer.js`
- Pipeline:
  1. `analyzeRepository(owner, repo)` orchestrates the deep pass
  2. `fetchRepositoryStructure` crawls repo and captures contents
  3. Purpose/domain: `analyzeAllFilesForPurpose` aggregates signals from `package.json`, README/docs, code/configs
  4. Architecture: `analyzeArchitecture` detects patterns, layers, design patterns, and structural quality
  5. Coherence: naming, structure, and pattern consistency scores
  6. Code quality: maintainability, readability, performance, security, testability
  7. `generateInsights` yields strengths, weaknesses, and concrete recommendations
- Key detectors:
  - Architecture pattern: Next.js App/Pages router, MVC, Layered, Express REST, Microservices, or Custom
  - Layer separation: Presentation, Business Logic, Data Access, Utilities, Configuration, Middleware
  - Design patterns: Singleton (config), Factory, Repository, Observer, Middleware, MVC, Service Layer, Component
  - Complexity: simple, moderate, complex (based on file/dir counts)

### 6) Presentation and results layer

- Files:
  - `menu.js`: Interactive CLI (Inquirer), template/hackathon selection, saving results
  - `index.js`: Minimal sample script
- Output:
  - Colored console sections (optional `chalk`)
  - Criteria breakdown and feedback
  - Intelligent analysis summary (purpose, architecture, coherence, quality)
  - Optional JSON export via `menu.js` → `saveResultsToFile`

## Data flow

```mermaid
flowchart TD
  A[CLI input
  owner/repo
  template
  hackathon] --> B[fetchRepoData
  (metadata, contributors, commits)]
  A --> C[Template-driven analysis
  getComprehensiveCodeAnalysis]
  B --> C
  C --> D[Objective scores
  quality/docs/functionality/
  innovation/ux]
  B --> E[IntelligentProjectAnalyzer
  crawl + content analysis]
  E --> F[Intelligent results
  purpose/architecture/
  coherence/quality/insights]
  D --> G[Weighted final score]
  F --> H[Console output +
  optional JSON export]
  A --> I[Hackathon checks
  dates/team/original/demo]
  I --> H
  C --> J[LLM evals (optional)
  Together AI]
  J --> H
```

## Hackathon eligibility mode

- Config file: `hackathonTemplates.json`
- Checks performed (if a template is selected in the CLI):
  - Dates: repo `created_at` must be ≥ startDate, last push ≤ deadline
  - Team size: contributor count ≤ `maxTeamSize` (if provided)
  - Originality: fails if repo is a fork and `mustBeOriginal` is true
  - Demo: passes if any file name contains “demo” when `demoRequired` is true

Example entry:

```json
{
  "hederahacks": {
    "hackathonName": "HederaHacks",
    "startDate": "2025-07-01",
    "deadline": "2025-07-07",
    "maxTeamSize": 4,
    "mustBeOriginal": true,
    "demoRequired": true
  }
}
```

## Usage

### Interactive mode (recommended)

```bash
npm start
# Follow prompts to select owner/repo, hackathon (optional), and template (or auto-detect)
```

### Programmatic sample

Edit `index.js` to set your `owner` and `repo`, then:

```bash
npm run analyze
```

### Saving reports

In the CLI, enable “Save results to file” to create a timestamped JSON report in the project root.

## Configuration

- Environment variables
  - `GITHUB_TOKEN` (required): increases rate limits and enables content fetch
  - `TOGETHER_API_KEY` (optional but recommended): enables LLM evaluations
- Rate limits: Large repos may throttle; the analyzer includes timeouts and skips large/binary files

## Extending the system

### Add a new project template

1. Edit `projectTemplates.json`
2. Provide a `description`, `files` (with optional wildcards), and `criteria.fileStructure`
3. Run the CLI and select your new template

### Tune scoring weights

Adjust `CRITERIA_WEIGHTS` in `evaluateRepo.js`

### Add new objective metrics

- Implement new functions in `evaluateRepo.js` and aggregate into the final score

### Enhance intelligent analysis

- Extend detectors in `intelligentAnalyzer.js` (architecture patterns, layers, design patterns, quality heuristics)

## File map (high level)

- `menu.js`: Interactive CLI, results saving
- `evaluateRepo.js`: Orchestration, GitHub fetch, template/static analysis, objective and AI scoring, hackathon checks
- `intelligentAnalyzer.js`: Deep semantic analyzer (purpose, architecture, coherence, quality, insights)
- `projectTemplates.json`: Project-type expectations and criteria
- `hackathonTemplates.json`: Hackathon rules and windows
- `index.js`: Minimal scripted run example
- `githubAPI.js`: Basic GitHub fetch helper (not used in main flow)

## Troubleshooting

- Missing or low results:
  - Ensure `GITHUB_TOKEN` is set and valid
  - Private repos require a token with repo scope
- LLM timeouts:
  - Calls are timeboxed; failures fall back to defaults
  - Verify `TOGETHER_API_KEY` and network connectivity
- Large repos:
  - The crawler skips large/binary files and some build directories by design

## License

ISC
