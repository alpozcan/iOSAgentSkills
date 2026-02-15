# On-Device LLM Integration with Apple Foundation Models

## Context
You're building an iOS app (iOS 26+) that uses AI for conversational features, and you need 100% on-device inference — no data leaves the device. Apple's Foundation Models framework provides the system LLM pre-installed on device, with no model download, no API key, and OS-managed memory/power budgets.

## Pattern

### Protocol-First LLM Engine

```swift
public protocol LLMEngineProtocol: Sendable {
    func generateResponse(prompt: String) async throws -> String
    func generateResponse(systemPrompt: String, userPrompt: String) async throws -> String
    func generateStream(prompt: String) -> AsyncThrowingStream<String, Error>
    func generateStream(systemPrompt: String, userPrompt: String) -> AsyncThrowingStream<String, Error>
    func isAvailable() async -> Bool
}

// Default implementations for backward compat
public extension LLMEngineProtocol {
    func generateResponse(systemPrompt: String, userPrompt: String) async throws -> String {
        try await generateResponse(prompt: userPrompt)
    }
}
```

### Apple Foundation Models Integration

```swift
import FoundationModels

public final class FoundationModelsEngine: LLMEngineProtocol, @unchecked Sendable {
    private let dynamicPromptEngine: DynamicPromptEngine
    
    public func generateResponse(systemPrompt: String, userPrompt: String) async throws -> String {
        let session = LanguageModelSession(instructions: systemPrompt)
        let response = try await session.respond(to: userPrompt)
        return response.content
    }
    
    public func generateStream(systemPrompt: String, userPrompt: String) -> AsyncThrowingStream<String, Error> {
        AsyncThrowingStream { continuation in
            Task {
                do {
                    let session = LanguageModelSession(instructions: systemPrompt)
                    let stream = session.streamResponse(to: userPrompt)
                    for try await partial in stream {
                        continuation.yield(partial.content)
                    }
                    continuation.finish()
                } catch {
                    continuation.finish(throwing: error)
                }
            }
        }
    }
    
    public func isAvailable() async -> Bool {
        let session = LanguageModelSession()
        do {
            let _ = try await session.respond(to: "test")
            return true
        } catch { return false }
    }
}
```

### Dynamic Prompt Gene Pool System

Zero hardcoded prompts. All prompts are assembled at runtime from composable "genes":

```swift
public struct PromptGene: Codable, Identifiable, Sendable {
    public let id: UUID
    public let type: GeneType
    public var content: String
    public var version: Int
    public var fitnessScore: Float        // 0.0–1.0, used for selection
    public var parentGeneId: UUID?        // Lineage tracking
    public var positiveReactions: Int
    public var negativeReactions: Int
    public var evolutionDirective: String  // How this gene should mutate
    
    public var acceptanceRatio: Float {
        let total = positiveReactions + negativeReactions
        guard total > 0 else { return 0.5 }
        return Float(positiveReactions) / Float(total)
    }
}

public enum GeneType: String, Codable, CaseIterable, Sendable {
    case systemPersona        // "You are Wythnos..."
    case responseFormat       // Bullet points, time formats
    case domainInstruction    // Schedule queries, conflict detection
    case contextTemplate      // Calendar context injection with placeholders
    case evolutionDirective   // How to mutate
    case insightPattern       // What patterns to look for
    case emotionalTone        // Warm, concise, etc.
    case languageMixing       // Turkish/English handling
    case errorRecovery        // Graceful failures
    case safetyGuardrail      // Content policy
}
```

### Prompt Synthesis (Gene Selection → Assembly)

```swift
public func synthesizePrompt(for intent: UserIntent, context: CalendarSnapshot, ...) -> SynthesizedPrompt {
    var components: [PromptGene] = []
    
    // 1. Select best gene for each slot (fitness-weighted random selection)
    if let persona = genePool.selectBestGene(type: .systemPersona) { components.append(persona) }
    if let format = genePool.selectBestGene(type: .responseFormat) { components.append(format) }
    if let instruction = genePool.selectBestGene(type: .domainInstruction, for: intent.category) {
        components.append(instruction)
    }
    
    // 2. Fill context template with real calendar data
    if let contextGene = genePool.selectBestGene(type: .contextTemplate) {
        let filled = fillContextTemplate(contextGene, with: context)
        components.append(filled)
    }
    
    // 3. Add tone, error recovery, evolution directives
    // ...
    
    return SynthesizedPrompt(components: components, metadata: metadata)
}
```

### Evolution: Feedback → Fitness → Mutation

```swift
public func evolveFromFeedback(prompt: SynthesizedPrompt, response: String, feedback: UserFeedback) {
    for gene in prompt.components {
        if feedback.isPositive {
            genePool.adjustFitness(gene.id, delta: +0.05)
            genePool.recordPositiveReaction(gene.id)
        } else {
            genePool.adjustFitness(gene.id, delta: -0.08)
            genePool.recordNegativeReaction(gene.id)
            
            // Low fitness triggers mutation
            if gene.fitnessScore < 0.3 && gene.usageCount > 10 {
                let mutated = genePool.mutateGene(gene, feedback: feedback)
                genePool.addGene(mutated)
            }
        }
    }
}
```

### Safety Layer (Decorator Pattern)

```swift
public actor SafetyAwareLLMEngine: LLMEngineProtocol {
    private let baseEngine: LLMEngineProtocol
    private let safetyGuardrail: SafetyGuardrailProtocol
    private let contentFilter: ContentSafetyFilter
    
    public func generateResponse(systemPrompt: String, userPrompt: String) async throws -> String {
        let classification = await safetyGuardrail.classify(userPrompt, locale: .current)
        
        switch classification {
        case .safe:
            let safeSystemPrompt = injectSafetyInstructions(systemPrompt)
            let response = try await baseEngine.generateResponse(systemPrompt: safeSystemPrompt, userPrompt: userPrompt)
            return try await filterAndValidate(response)
        case .offTopic:
            return safetyGuardrail.generateRefusal(for: .offTopic, locale: .current)
        case .harmful(let type):
            return safetyGuardrail.generateRefusal(for: type, locale: .current)
        case .emergency:
            return generateEmergencyResponse()  // Crisis resources
        }
    }
}
```

### Response Sanitizer (Output Hygiene)

```swift
public struct ResponseSanitizer: ResponseSanitizerProtocol {
    public func sanitize(_ response: String) -> SanitizedResponse {
        var cleaned = response
        var followUp: String?
        
        // 1. Extract follow-up questions
        if let range = cleaned.range(of: "Follow-up question:", options: .caseInsensitive) {
            followUp = String(cleaned[range.upperBound...]).trimmed
            cleaned = String(cleaned[..<range.lowerBound])
        }
        
        // 2. Strip meta-labels ("Most useful part:", "Ignored information:")
        for pattern in labelPatterns {
            cleaned = cleaned.replacingOccurrences(of: pattern, with: "", options: .caseInsensitive)
        }
        
        // 3. Fix whitespace artifacts
        // 4. Fix missing spaces after punctuation
        
        return SanitizedResponse(mainContent: cleaned, followUpQuestion: followUp, strippedLabels: stripped)
    }
}
```

## Why This Matters

- **Zero data leaves the device** — all inference is on-device, perfect for privacy-sensitive domains
- **No model download** — Apple's Foundation Models are pre-installed with the OS
- **Gene pool evolution** — prompts improve over time based on user feedback, without retraining
- **Safety layer as decorator** — `SafetyAwareLLMEngine` wraps `FoundationModelsEngine` transparently
- **Response sanitization** catches LLM artifacts (meta-labels, missing spaces) before they reach the UI

## Anti-Patterns

- Don't hardcode prompts as string literals — use the gene pool system
- Don't skip the safety classification layer — even on-device models can produce harmful content
- Don't assume `isAvailable()` always returns true — device may lack Neural Engine
- Don't accumulate `LanguageModelSession` instances — create them per request
- Don't display raw LLM output — always sanitize first
