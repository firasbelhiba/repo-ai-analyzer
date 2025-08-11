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

- Implementation details:
  - API endpoints: `GET /repos/{owner}/{repo}/contents/{path}` recursively for directory listings and file blobs.
  - Directory skip list: `node_modules`, `.git`, `build`, `dist`, `out`, `.next`, `coverage`, `.nyc_output`, `.cache`, `tmp`, `temp`, `vendor`, `bower_components`, `.pnp`.
  - File skip rules: binary and build artifacts (e.g., images/fonts/archives/executables) and any file > 1MB are ignored.
  - “Important files” heuristic: extensions `.js/.jsx/.ts/.tsx/.json/.md/.txt/.yml/.yaml/.env` and known names like `package.json`, `README.md`, `Dockerfile`, `docker-compose.yml`, `.gitignore` are stored in `structure.files` for fast access.
  - Structure model populated: `{ files, directories, allFiles, allDirectories, fileContents, analysis }` for downstream consumers.
  - Resiliency: directory and file fetch errors are handled non-fatally, logged with warnings, and crawling continues.
  - Rate limits: honors GitHub API rate-limiting via `GITHUB_TOKEN`; graceful degradation if some files cannot be fetched.

### 2) Template-driven static analysis layer

- Files:
  - `projectTemplates.json`: Catalog of project types (Next.js, React, Node.js, Fullstack, Mobile, Custom) with expected files, directories, and evaluation criteria.
  - `evaluateRepo.js` → `getComprehensiveCodeAnalysis(repoData, selectedTemplate)`
- Responsibilities:
  - Auto-detect project type (`detectProjectTypeFromRepo`) or use user-selected template
  - Expand template wildcards (e.g., `pages/api/*`) and fetch content for those files
  - Compute per-file stats (size, line count, type) and a project summary (file count, line count, file-type distribution)

- Implementation details:
  - Project auto-detection combines root listing + `package.json` dependency inspection (e.g., `next` → Next.js, `react` → React, presence of `index.js`/`server.js` → Node.js).
  - Wildcards are directory-expanded at fetch time; only files are analyzed from those directories.
  - Essential files (e.g., `package.json`, `index.js`, `server.js`, `app.js`, `App.js`, `App.tsx`) are added if not already included by the template to improve coverage.
  - Summary aggregates: total files analyzed, total lines, and file-type frequency map for quick at-a-glance metrics.
  - Robustness: missing files are logged and skipped without failing the run.

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

- Scoring mechanics (high level):
  - Project structure (up to 30 pts): compares `projectTemplates.json` criteria vs. discovered files; awards per expected file, with bonus for complete categories.
  - Code quality (up to 25 pts): comment ratio, consistent indentation, modern language features (e.g., `async/await`, `const/let`, TS types), and presence of error handling.
  - Security & performance (up to 25 pts): env usage, security middleware/packages (helmet/cors/rate-limit), and performance deps (compression/cache/redis), plus testing setup.
  - Modern practices (up to 20 pts): build tools (Vite/Webpack/Tailwind/TS), CI/CD config, containerization, and database integration indicators.
  - Functionality checks: database connection, routes, auth hints, presence of routes/controllers/models directories.
  - Documentation: README presence and structure (headings), installation steps, and API references.
  - UX: API docs and usage examples in README, explicit error handling patterns, CORS configuration.
  - All raw category points are scaled to 0–10 for reporting and then weighted by `CRITERIA_WEIGHTS`.

### 4) LLM-assisted evaluation layer

- Files/Functions (in `evaluateRepo.js`):
  - Together AI client: `together-ai`
  - Timeboxed calls via `callTogetherAIWithTimeout`
  - AI judgments: `evaluateCodeQualityWithAI`, `evaluateDocumentationWithAI`, `evaluateFunctionalityWithAI`, `evaluateInnovationWithAI`, `evaluateUserExperienceWithAI`
  - Optional Hackathon Feasibility: Prompted summary for “could this be built within the window?”
- Responsibilities:
  - Produce concise expert assessments and an extracted numeric score
  - Fail-safe defaults if timeouts/errors occur

- Implementation details:
  - Model: `deepseek-ai/DeepSeek-V3` via Together AI, 30s timeout per call with Promise.race-based cancellation.
  - Inputs: a representative code sample and/or README text are provided depending on the metric.
  - Score extraction: `evaluateAIResponse` parses explicit “X/10” or “Score: X” patterns; falls back to keyword heuristics otherwise.
  - Robustness: timeouts or API errors return safe default scores with explanatory feedback.
  - Feasibility prompt (hackathon mode): summarizes codebase stats, contributor/commit counts, and the event window; requests a YES/NO plus reasoning.

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

- Heuristics and metrics in depth:
  - Purpose extraction:
    - `package.json` dependencies map to technologies (e.g., `express`, `react`, `next`, `mongoose`, `prisma`, `typescript`, `tailwindcss`, `jest`, `cypress`).
    - README keywords infer domain (finance/e-commerce/social/task/backend).
    - Codefile scan looks for purpose hints (“budget”, “finance”, “transaction”, “auth”, “cron”).
  - Architecture detection:
    - Directory cues (e.g., `app/` vs `pages/` for Next.js, `controllers/models/views` for MVC, `services/controllers` for layered, Docker indicators for microservices).
    - File organization report: root vs. config vs. docs vs. testing vs. deployment; `src/` breakdown (components/services/utils/types/hooks/pages/assets).
  - Layer separation: checks for presence of typical layer directories and returns `[Presentation, Business Logic, Data Access, Utilities, Configuration, Middleware]` subset or `Monolithic`.
  - Coherence scoring (0–10):
    - Naming consistency: casing conventions across file/dir names, dominant patterns ratio.
    - Structural consistency: directory depth variance, presence of logical groupings (components/services/utils/config) and occupancy of directories.
    - Pattern consistency: dominant extensions ratio, grouping of similar file types, config grouping.
  - Quality sub-scores (0–10 each):
    - Maintainability: modular dirs, separation of concerns, config presence, documentation presence.
    - Readability: docs presence, clear directory names, organization level, comment presence within fetched files.
    - Performance: build/optimization configs and packages (`webpack`/`vite`, `compression`, caching).
    - Security: security-related files and packages (`helmet`, `cors`, `bcrypt`, `jsonwebtoken`, `express-rate-limit`).
    - Testability: test files/dirs and packages (`jest`, `mocha`, `cypress`, `playwright`, `@testing-library`).
  - Insights and recommendations:
    - `generateInsights` classifies outputs into categories (understanding, strength, improvement, information) with confidence.
    - `generateSpecificRecommendations` suggests actionable steps for documentation, architecture, consistency, testing, security, and configuration.

### 6) Presentation and results layer

- Files:
  - `menu.js`: Interactive CLI (Inquirer), template/hackathon selection, saving results
  - `index.js`: Minimal sample script
- Output:
  - Colored console sections (optional `chalk`)
  - Criteria breakdown and feedback
  - Intelligent analysis summary (purpose, architecture, coherence, quality)
  - Optional JSON export via `menu.js` → `saveResultsToFile`

- CLI flow details:
  - Prompts for `owner` and `repo`, hackathon selection (or skip), project template (or auto-detect), and whether to show a detailed scoring breakdown.
  - Optional: view all project templates and key files before running analysis.
  - Save results option writes a timestamped JSON file (e.g., `analysis_owner_repo_YYYY-MM-DDTHH-mm-ssZ.json`).
  - After results, you can analyze another repo, view templates again, or exit.

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
