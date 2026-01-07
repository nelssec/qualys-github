# Qualys GitHub Actions - Architecture & Design

## Why We Built This

Modern software development teams face an increasing challenge: shipping code quickly while ensuring security. Vulnerabilities in container images and third-party dependencies are among the top causes of security breaches. Traditional security scanning often happens too late in the development cycle, creating friction between security and development teams.

We built Qualys GitHub Actions to solve this by:

1. **Shifting Security Left** - Catch vulnerabilities during pull requests, not after deployment
2. **Developer-First Experience** - Results appear directly in GitHub where developers work
3. **Organization-Wide Consistency** - Deploy once, protect every repository
4. **Actionable Feedback** - Automatic issue creation and clear pass/fail criteria

## How It Works

### High-Level Architecture

```mermaid
flowchart TB
    subgraph GitHub["GitHub Organization"]
        subgraph Repo1["Repository A"]
            WF1[Workflow] --> CS1[Code Scan]
            WF1 --> CT1[Container Scan]
        end
        subgraph Repo2["Repository B"]
            WF2[Workflow] --> CS2[Code Scan]
        end
        subgraph Repo3["Repository C"]
            WF3[Workflow] --> CT3[Container Scan]
        end

        OrgSecrets[(Org Secrets\nQUALYS_ACCESS_TOKEN)]
        OrgVars[(Org Variables\nQUALYS_POD)]
    end

    subgraph Actions["Qualys GitHub Actions"]
        CodeScan[Code Scan Action]
        ContainerScan[Container Scan Action]
        QScanner[QScanner Binary]
    end

    subgraph Qualys["Qualys Cloud"]
        API[Container Security API]
        Policies[Security Policies]
        VulnDB[(Vulnerability Database)]
    end

    subgraph Outputs["Results"]
        SARIF[SARIF Report]
        SecurityTab[GitHub Security Tab]
        Issues[GitHub Issues]
    end

    CS1 --> CodeScan
    CT1 --> ContainerScan
    CS2 --> CodeScan
    CT3 --> ContainerScan

    OrgSecrets -.-> WF1
    OrgSecrets -.-> WF2
    OrgSecrets -.-> WF3
    OrgVars -.-> WF1
    OrgVars -.-> WF2
    OrgVars -.-> WF3

    CodeScan --> QScanner
    ContainerScan --> QScanner

    QScanner <--> API
    API <--> Policies
    API <--> VulnDB

    QScanner --> SARIF
    SARIF --> SecurityTab
    SARIF --> Issues
```

### Scan Flow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant Action as Qualys Action
    participant QS as QScanner
    participant Qualys as Qualys Cloud

    Dev->>GH: Push code / Open PR
    GH->>Action: Trigger workflow

    Action->>Action: Download QScanner binary
    Action->>Action: Verify SHA256 checksum

    alt Code Scan
        Action->>QS: Scan repository files
        QS->>QS: Detect dependencies
        QS->>Qualys: Upload manifest
        Qualys->>Qualys: Match against vuln DB
        Qualys-->>QS: Return vulnerabilities
    else Container Scan
        Action->>QS: Scan container image
        QS->>QS: Extract image layers
        QS->>Qualys: Upload scan data
        Qualys->>Qualys: Analyze packages
        Qualys-->>QS: Return vulnerabilities
    end

    QS-->>Action: SARIF report

    Action->>GH: Upload SARIF to Security tab

    alt Threshold Check
        Action->>Action: Compare against thresholds
    else Policy Check
        Action->>Qualys: Evaluate policy
        Qualys-->>Action: ALLOW/DENY/AUDIT
    end

    alt Vulnerabilities Found
        Action->>GH: Create Issues (optional)
    end

    Action-->>GH: Set pass/fail status
    GH-->>Dev: Show results in PR
```

### Pass/Fail Decision Logic

```mermaid
flowchart TD
    Start[Scan Complete] --> Mode{Evaluation Mode?}

    Mode -->|Policy| PolicyEval[Evaluate Qualys Policy]
    Mode -->|Threshold| ThresholdEval[Check Thresholds]

    PolicyEval --> PolicyResult{Policy Result}
    PolicyResult -->|ALLOW| Pass[PASS]
    PolicyResult -->|DENY| Fail[FAIL]
    PolicyResult -->|AUDIT| Warn[WARN - Continue]

    ThresholdEval --> CritCheck{Critical > Max?}
    CritCheck -->|Yes| Fail
    CritCheck -->|No| HighCheck{High > Max?}
    HighCheck -->|Yes| Fail
    HighCheck -->|No| MedCheck{Medium > Max?}
    MedCheck -->|Yes| Fail
    MedCheck -->|No| LowCheck{Low > Max?}
    LowCheck -->|Yes| Fail
    LowCheck -->|No| Pass

    Pass --> Continue{continue_on_error?}
    Fail --> Continue
    Warn --> Output

    Continue -->|true| Output[Set Outputs]
    Continue -->|false & FAIL| FailWorkflow[Fail Workflow]

    Output --> End[End]
    FailWorkflow --> End
```

## Organization-Wide Deployment

### Setup Flow

```mermaid
flowchart LR
    subgraph Setup["One-Time Setup"]
        A[Create Org Secret] --> B[Create Org Variable]
        B --> C[Add Reusable Workflow]
    end

    subgraph Repos["Each Repository"]
        D[Add workflow file] --> E[Reference reusable workflow]
    end

    Setup --> Repos

    style Setup fill:#e1f5fe
    style Repos fill:#f3e5f5
```

### Three Deployment Options

```mermaid
flowchart TB
    subgraph Option1["Option 1: Direct Action"]
        R1[Repository] --> A1[uses: qualys/qualys-github/code-scan@v1]
        A1 --> Full1[Full configuration in each repo]
    end

    subgraph Option2["Option 2: Reusable Workflow"]
        R2[Repository] --> W2[uses: .github/workflows/qualys-code-scan.yml]
        W2 --> Shared2[Shared configuration]
    end

    subgraph Option3["Option 3: Starter Workflow"]
        R3[New Repository] --> New3[Actions → New Workflow]
        New3 --> Template3[Select Qualys template]
        Template3 --> Auto3[Auto-configured]
    end

    style Option1 fill:#ffebee
    style Option2 fill:#e8f5e9
    style Option3 fill:#fff3e0
```

## How It Helps Development Teams

### Vulnerability Lifecycle

```mermaid
flowchart LR
    subgraph Before["Without Qualys Actions"]
        B1[Write Code] --> B2[Merge to Main]
        B2 --> B3[Deploy to Prod]
        B3 --> B4[Security Scan]
        B4 --> B5[Find Vulnerabilities]
        B5 --> B6[Emergency Fix]
        B6 --> B7[Redeploy]
    end

    subgraph After["With Qualys Actions"]
        A1[Write Code] --> A2[Open PR]
        A2 --> A3[Auto Scan]
        A3 --> A4{Vulnerabilities?}
        A4 -->|Yes| A5[Fix in PR]
        A5 --> A3
        A4 -->|No| A6[Merge Safely]
        A6 --> A7[Deploy with Confidence]
    end

    style Before fill:#ffcdd2
    style After fill:#c8e6c9
```

### Integration Points

```mermaid
flowchart TB
    subgraph Triggers["Workflow Triggers"]
        Push[Push to Branch]
        PR[Pull Request]
        Schedule[Scheduled Scan]
        Manual[Manual Dispatch]
    end

    subgraph Scan["Qualys Scan"]
        Action[GitHub Action]
    end

    subgraph Results["Result Destinations"]
        Security[GitHub Security Tab]
        Issues[GitHub Issues]
        Artifacts[Workflow Artifacts]
        SBOM[SBOM Files]
    end

    subgraph Decisions["Workflow Control"]
        Gate[Quality Gate]
        Block[Block Merge]
        Notify[Notify Team]
    end

    Push --> Action
    PR --> Action
    Schedule --> Action
    Manual --> Action

    Action --> Security
    Action --> Issues
    Action --> Artifacts
    Action --> SBOM

    Action --> Gate
    Gate -->|Fail| Block
    Gate -->|Pass/Warn| Notify
```

## Key Benefits

| Benefit | Description |
|---------|-------------|
| **Shift Left** | Find vulnerabilities during development, not after deployment |
| **Developer Experience** | Results in GitHub Security tab, automatic issue creation |
| **Org-Wide Coverage** | Single configuration protects all repositories |
| **Flexible Policies** | Local thresholds or centralized Qualys policies |
| **SBOM Generation** | Automatic software bill of materials for compliance |
| **Native Integration** | SARIF format for GitHub code scanning alerts |

## Security Model

```mermaid
flowchart TB
    subgraph Secrets["Secret Management"]
        OrgSecret[Org-Level Secret]
        RepoSecret[Repo-Level Secret]
        EnvSecret[Environment Secret]
    end

    subgraph Runtime["Runtime Security"]
        Mask[Token Masking]
        HTTPS[HTTPS Only]
        Checksum[SHA256 Verification]
    end

    subgraph Data["Data Flow"]
        Scan[Scan Data]
        Results[Results]
    end

    OrgSecret --> Mask
    RepoSecret --> Mask
    EnvSecret --> Mask

    Mask --> Scan
    Scan -->|Encrypted| HTTPS
    HTTPS --> Results

    Checksum -.->|Verify Binary| Scan
```

The Qualys GitHub Actions are designed with security as a first principle:

- Access tokens are never logged and always masked
- Binary downloads are verified with SHA256 checksums
- All API communication uses HTTPS
- Results stay within your GitHub organization
