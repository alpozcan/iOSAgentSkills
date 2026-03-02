---
title: "On-Device LLM Integration with Apple Foundation Models"
description: "100% on-device LLM using Apple Foundation Models with dynamic prompt gene pool, fitness-weighted selection, feedback-driven evolution, safety classification decorator, and response sanitization."
---

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
// Calendar context comes from [[12-eventkit-coredata-sync-architecture]]
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
/// Step 1: Adjust fitness scores based on user feedback
public func applyFeedbackToFitness(prompt: SynthesizedPrompt, feedback: UserFeedback) {
    for gene in prompt.components {
        if feedback.isPositive {
            genePool.adjustFitness(gene.id, delta: +0.05)
            genePool.recordPositiveReaction(gene.id)
        } else {
            genePool.adjustFitness(gene.id, delta: -0.08)
            genePool.recordNegativeReaction(gene.id)
        }
    }
    triggerMutationsIfNeeded(prompt: prompt, feedback: feedback)
}

/// Step 2: Mutate low-fitness genes (separated for testability)
public func triggerMutationsIfNeeded(prompt: SynthesizedPrompt, feedback: UserFeedback) {
    guard !feedback.isPositive else { return }
    for gene in prompt.components where gene.fitnessScore < 0.3 && gene.usageCount > 10 {
        let mutated = genePool.mutateGene(gene, feedback: feedback)
        genePool.addGene(mutated)
    }
}
```

### Safety Layer (Decorator Pattern)

```swift
// Actor isolation — see [[06-actor-based-concurrency-patterns]]
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
            // Refusal responses are [[16-localization-and-multi-language-patterns|localized]]
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

## Edge Cases

- **Gene pool depletion:** If all genes for a type fall below fitness 0.3, selection returns nil and prompt synthesis produces incomplete prompts. Mitigation: set a fitness floor (e.g., 0.15) so genes never fully die, or maintain immutable "seed genes" that reset fitness periodically.
- **Foundation Models unavailability:** Devices without Apple Neural Engine (or running older iOS) won't have `FoundationModels`. Always check `isAvailable()` before inference and fall back to cached/template responses. Gate this at the ViewModel level, not inside the engine.
- **Concurrent mutation from simultaneous feedback:** Multiple feedback events arriving concurrently can trigger duplicate mutations of the same gene. Use the gene pool's actor isolation to serialize feedback processing.
- **Response sanitizer stripping valid content:** Aggressive label stripping (e.g., removing "Follow-up question:") could match legitimate user-facing content. Use anchored patterns (start-of-line only) and test against diverse LLM outputs.
- **Session lifecycle:** `LanguageModelSession` should be created per-request, not reused. Reusing sessions accumulates context and can degrade response quality or hit memory limits.
- **Empty response handling:** The Foundation Models API may return an empty string for certain prompts. Check `response.content.isEmpty` and either retry with a rephrased prompt or return a graceful fallback message.
- **Regex in sanitizer:** The `labelPatterns` used in `ResponseSanitizer` must avoid catastrophic backtracking. Use simple literal prefix matches or `String.hasPrefix()` instead of complex regex. See security notes on ReDoS prevention.

## Why This Matters

- **Zero data leaves the device** — all inference is on-device, perfect for privacy-sensitive domains (tracked via [[10-privacy-first-analytics-architecture|privacy-first analytics]])
- **No model download** — Apple's Foundation Models are pre-installed with the OS
- **Gene pool evolution** — prompts improve over time based on user feedback, without retraining
- **Safety layer as decorator** — `SafetyAwareLLMEngine` wraps `FoundationModelsEngine` transparently, with [[14-error-handling-and-typed-error-system|typed errors]] for classification failures
- **Response sanitization** catches LLM artifacts (meta-labels, missing spaces) before they reach the UI

## Anti-Patterns

- Don't hardcode prompts as string literals — use the gene pool system
- Don't skip the safety classification layer — even on-device models can produce harmful content
- Don't assume `isAvailable()` always returns true — device may lack Neural Engine. [[07-storekit2-intelligence-based-trial|Query limits]] gate access during the free tier
- Don't accumulate `LanguageModelSession` instances — create them per request
- Don't display raw LLM output — always sanitize first
- Don't use complex regex for response sanitization — prefer `String` prefix/suffix operations to avoid ReDoS vulnerabilities
- Don't reuse `LanguageModelSession` across requests — create fresh sessions to avoid context accumulation
