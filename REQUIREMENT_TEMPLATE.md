# Requirement Template - Web & API Applications

## Metadata
- **ID**: REQ-[MODULE]-[###] (e.g., REQ-AUTH-001)
- **Title**: [Concise requirement name]
- **Version**: [Semantic version, e.g., 1.0.0]
- **Status**: [Draft | Review | Approved | Deprecated]
- **Author**: [Name]
- **Last Updated**: [ISO 8601 date]
- **Priority**: [Critical | High | Medium | Low]
- **Classification**: [Functional | Security | Compliance | Performance | Operational]

## Requirement
- **Description**: [Single-sentence, unambiguous statement of what the system MUST do. Use RFC 2119 keywords: MUST, MUST NOT, SHALL, SHOULD, MAY.]
- **Rationale**: [Why this requirement exists - business, regulatory, or threat-driven justification.]

## Scope
- **Applies To**: [Web App | API | Both]
- **Components**: [e.g., Authentication Service, Transaction API, Account Portal]
- **Actors**: [e.g., Authenticated Customer, Service Account, Internal Admin, Third-Party Aggregator]
- **Data Classification**: [Public | Internal | Confidential | Restricted/PII | Regulated (PCI-DSS, SOX, GLBA)]

## Security Context
- **Defense Layer**: [Architecture | Input Validation | Strict API | Encoding | Sanitization]
- **Threat(s) Addressed**: [Reference specific threats, e.g., OWASP Top Ten A01:2021, CWE-89, STRIDE category]
- **Trust Boundary**: [Where does this requirement enforce a trust boundary? e.g., API gateway, service mesh, client-server edge]
- **Zero Trust Consideration**: [How does this requirement treat input as untrusted? What independent validation does it perform?]

## Standards Alignment
- **OWASP ASVS**: [Control ID, e.g., V2.1.1]
- **OWASP AISVS**: [Control ID, if AI/LLM components involved]
- **NIST SP 800-53**: [Control family and ID, e.g., SI-10, AC-3]
- **NIST SP 800-207**: [Zero Trust principle, if applicable]
- **Regulatory**: [PCI-DSS requirement, GLBA/Reg P section, SOX control, OCC guidance, FFIEC reference]
- **Other**: [RFC 9700, FIDO2/WebAuthn, ISO 27001 control, etc.]

## Acceptance Criteria
[Each criterion MUST be independently testable. Use Given/When/Then or explicit pass/fail conditions.]

1. **AC-01**: Given [precondition], when [action], then [expected outcome].
2. **AC-02**: Given [precondition], when [invalid/malicious input], then [system rejects input and returns appropriate error - no processing occurs].
3. **AC-03**: [Negative test case - what MUST NOT happen.]

## Failure Behavior
- **On Invalid Input**: [Reject with HTTP [status code], log event with correlation ID, do NOT disclose internal state.]
- **On System Error**: [Fail closed | Fail open (requires explicit justification) | Circuit breaker behavior]
- **Alerting**: [What triggers an alert? Threshold? Destination?]

## Test Strategy
- **Unit Tests**: [What logic is unit-testable? Target coverage.]
- **Integration Tests**: [Cross-service or cross-boundary validation.]
- **Security Tests**: [SAST rule, DAST scan target, fuzzing input class, manual pentest checklist item.]
- **Compliance Tests**: [Automated evidence collection for audit - log presence, config validation, policy enforcement.]
- **Coverage Target**: [>= 80% branch coverage for implementing module]

## Dependencies
- **Upstream**: [Requirements this depends on, e.g., REQ-AUTH-001 must be satisfied before REQ-TXN-003]
- **Downstream**: [Requirements that depend on this one]
- **External**: [Third-party services, HSMs, identity providers, core banking APIs]

## Implementation Notes
- **Constraints**: [Technology constraints, latency budgets, regulatory deadlines]
- **Anti-Patterns**: [Explicitly prohibited approaches - e.g., "MUST NOT use client-side validation as sole enforcement", "MUST NOT log raw PAN or credentials"]
- **AI Development Guidance**: [If AI-assisted development is used - reference applicable prompt library, required CLAUDE.md rules, mandatory human review gates]
