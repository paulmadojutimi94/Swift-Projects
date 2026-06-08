# SupportSense
SupportSense is a Swift-first prototype for AI-assisted support operations. The app helps teams ingest support conversations, generate concise summaries, surface SLA risk, recommend next actions, and capture quality insights in a clean Apple-platform-native workflow.

## Overview

SupportSense exists to demonstrate four engineering themes in one repo:

- AI-powered product features
- Internal-tool thinking
- User-friendly Apple-platform UI
- Testing and quality discipline

The project is intentionally small, modular, and easy to demo. It is designed for interview portfolios, rapid prototyping, and iterative extension.

## Features

- Case summarization from seeded or imported conversation data
- Priority and SLA risk signals
- Recommended next actions
- Search, filters, and saved views
- Local fixtures for offline demos
- Protocol-based AI service layer
- Unit tests and UI tests
- Clean, interview-friendly architecture

## Architecture

SupportSense uses a modular MVVM + use-case architecture.

```text
SupportSense/
├── App/
│   ├── SupportSenseApp.swift
│   └── Routing/
├── Features/
│   ├── CaseList/
│   ├── CaseDetail/
│   ├── Summary/
│   └── QualityInsights/
├── Domain/
│   ├── Models/
│   ├── UseCases/
│   └── Protocols/
├── Data/
│   ├── Network/
│   ├── Persistence/
│   ├── AI/
│   └── Fixtures/
├── Shared/
│   ├── DesignSystem/
│   ├── Extensions/
│   └── Utilities/
├── Tests/
└── UITests/
```

### Design choices

- **SwiftUI** for Apple-platform-native UI and fast iteration
- **Swift concurrency** for asynchronous AI and data operations
- **Protocol-driven services** for easy mocking and testability
- **Small feature modules** for maintainability
- **AI gateway abstraction** so the app can switch between:
  - a mock summarizer
  - a remote model endpoint
  - a future on-device adapter

## Setup

### Prerequisites

- Latest stable Xcode
- Latest stable Swift toolchain supported by that Xcode
- Git
- iOS Simulator or macOS target

### Clone and configure

```bash
git clone <your-repo-url>
cd SupportSense
cp Config/Example.xcconfig Config/Local.xcconfig
```

Update the local config with your development values:

```text
SUPPORTSENSE_API_BASE_URL = https://example.internal.api
SUPPORTSENSE_API_KEY = your-dev-key
```

Then open the project in Xcode and run the `SupportSense` scheme.

## Running the app

1. Launch the app.
2. Load seeded cases from local fixtures.
3. Select a case from the list.
4. Tap **Generate Summary**.
5. Review summary, urgency, next actions, and quality notes.
6. Save the generated note for later review.

## Usage

### Generate a summary

- Open a seeded case
- Tap **Generate Summary**
- Review the returned summary and recommended actions
- Save or copy the output

### Filter the queue

- Filter by priority
- Filter by overdue SLA risk
- Search by customer name, issue tag, or case ID
- Save a filtered view for repeated demos

## API examples

### Example request payload

```json
{
  "case_id": "CASE-1001",
  "locale": "en-IE",
  "conversation": "Customer cannot enroll device after password reset...",
  "metadata": {
    "product": "iPhone",
    "severity": "medium"
  }
}
```

### Example response payload

```json
{
  "summary": "Customer is blocked after a password reset and cannot complete device enrollment.",
  "priority": "high",
  "recommended_actions": [
    "Verify account state",
    "Confirm enrollment prerequisites",
    "Escalate if enrollment token is invalid"
  ],
  "quality_flags": [
    "Missing device OS version"
  ]
}
```

### cURL example

```bash
curl -X POST "$SUPPORTSENSE_API_BASE_URL/v1/summaries" \
  -H "Authorization: Bearer $SUPPORTSENSE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "case_id":"CASE-1001",
    "locale":"en-IE",
    "conversation":"Customer cannot enroll device after password reset...",
    "metadata":{"product":"iPhone","severity":"medium"}
  }'
```

## Swift code examples

### API client

```swift
import Foundation

struct SummaryRequest: Codable {
    let caseID: String
    let locale: String
    let conversation: String
}

struct SummaryResponse: Codable {
    let summary: String
    let priority: String
    let recommendedActions: [String]
}

protocol SummaryServing {
    func summarize(_ request: SummaryRequest) async throws -> SummaryResponse
}

final class SupportSenseClient: SummaryServing {
    private let baseURL: URL
    private let session: URLSession
    private let apiKey: String

    init(baseURL: URL, session: URLSession = .shared, apiKey: String) {
        self.baseURL = baseURL
        self.session = session
        self.apiKey = apiKey
    }

    func summarize(_ request: SummaryRequest) async throws -> SummaryResponse {
        var urlRequest = URLRequest(url: baseURL.appendingPathComponent("/v1/summaries"))
        urlRequest.httpMethod = "POST"
        urlRequest.addValue("application/json", forHTTPHeaderField: "Content-Type")
        urlRequest.addValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
        urlRequest.httpBody = try JSONEncoder().encode(request)

        let (data, response) = try await session.data(for: urlRequest)

        guard let http = response as? HTTPURLResponse, 200..<300 ~= http.statusCode else {
            throw URLError(.badServerResponse)
        }

        return try JSONDecoder().decode(SummaryResponse.self, from: data)
    }
}
```

### View model

```swift
import SwiftUI

@MainActor
final class CaseDetailViewModel: ObservableObject {
    @Published var summaryText = ""
    @Published var priority = ""
    @Published var isLoading = false
    @Published var errorMessage: String?

    private let service: SummaryServing

    init(service: SummaryServing) {
        self.service = service
    }

    func generateSummary(caseID: String, conversation: String) async {
        isLoading = true
        errorMessage = nil

        do {
            let result = try await service.summarize(
                SummaryRequest(
                    caseID: caseID,
                    locale: "en-IE",
                    conversation: conversation
                )
            )
            summaryText = result.summary
            priority = result.priority
        } catch {
            errorMessage = "Unable to generate summary. Please try again."
        }

        isLoading = false
    }
}
```

### Swift Testing example

```swift
import Testing
@testable import SupportSense

@Test("priority scoring prefers urgent cases")
func priorityScoringPrefersUrgentCases() {
    let urgent = PriorityScorer.score(slaMinutesRemaining: 5, negativeSentiment: true)
    let routine = PriorityScorer.score(slaMinutesRemaining: 240, negativeSentiment: false)

    #expect(urgent > routine)
}
```

## Contribution guide

Contributions are welcome if they improve clarity, testability, accessibility, or maintainability.

### Workflow

1. Create a feature branch:
   - `feature/case-summary`
   - `fix/empty-state`
   - `test/add-ui-regression-suite`
2. Keep pull requests small and focused.
3. Include tests for new behavior whenever practical.
4. Update documentation when architecture or setup changes.
5. Prefer descriptive names that read naturally in Swift.

### Code style

- Favor clear naming over clever naming
- Prefer protocol-based seams for dependencies
- Keep view models thin and business logic in use cases
- Handle loading, empty, and error states explicitly
- Design with accessibility in mind from the start

## Testing

SupportSense uses both unit tests and UI tests.

### Minimum quality bar

- Business rules covered by unit tests
- Core user flows covered by UI tests
- No crashes in the seeded demo path
- Error states demonstrated intentionally
- Accessibility labels added to interactive controls

## License

MIT

## Roadmap

- Add an on-device summarization adapter
- Add persisted saved views
- Add localization support
- Add a quality dashboard for repeated issue patterns
- Add richer fixture generation
- Add CI automation and pull-request quality checks
- Add architecture notes and onboarding docs
