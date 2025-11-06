# AFv2 Pattern #5: Looping

Validation-driven retry loop with automated testing and fix generation.

## Pattern Structure

```
Start → Generate → Validate → Gate → [PASS → Return | FIX → Fix Plan → loop back | FAIL → Return]
```

## Key Features

- 3-path gate (PASS, FIX, FAIL - not binary)
- Loop-back edge (Fix Plan → Generate, animated)
- Max 3 retries with automatic fix suggestions
- FAIL path prevents returning broken code

## Files

- `05-looping.json` - Complete Flowise workflow (858 lines)

## Quick Start

1. Import `05-looping.json` into Flowise
2. Configure Anthropic API key for all agents
3. Test with code generation + validation

## Use Cases

- Test-driven development (generate → test → fix loop)
- Policy compliance checking with remediation
- Automated code review with fixes
- Quality assurance workflows

## Documentation

See [Context Foundry Pattern Library](https://github.com/snedea/afv2-patterns-index) for complete documentation.

🤖 Built with Context Foundry
