# AFv2 Pattern #5: Looping - Test Cases

## Overview

**Pattern:** Validation-driven retry loop with automated testing and fix generation
**Flow:** Start → Generate → Validate → Gate → [PASS → Return | FIX → Fix Plan → loop back | FAIL → Return]
**Repository:** https://github.com/snedea/afv2-pattern-05-looping

---

## Prerequisites

### 1. Import Pattern into Flowise

1. Open Flowise UI (http://localhost:3000)
2. Navigate to **Agentflows** section
3. Click **"Add New"** → **"Import Agentflow"**
4. Upload `05-looping.json`
5. Pattern should load with nodes:
   - Start
   - Agent.Generate
   - Agent.Validate
   - Validation Gate (ConditionAgent)
   - Agent.FixPlan
   - Direct Reply (2 nodes: PASS, FAIL)
   - Sticky Notes (documentation)

### 2. Configure API Keys

**All agents require Anthropic API key configuration:**

1. Configure model for all agents:
   - **Credential:** Select your "Anthropic API Key" credential
   - **Model Name:** `claude-sonnet-4-5-20250929`
   - **Temperature:** `0.2`
   - **Streaming:** `true`

2. Save the workflow

### 3. Verify Node Configuration

**Expected Configuration:**

| Node | Type | State Updates | Key Feature |
|------|------|---------------|-------------|
| Agent.Generate | Agent | `loop.code`, `loop.attempt` | Code generation/fixing |
| Agent.Validate | Agent | `loop.validation_result`, `loop.errors` | Test execution |
| Validation Gate | ConditionAgent | Reads validation result | 3-path routing (PASS/FIX/FAIL) |
| Agent.FixPlan | Agent | `loop.fix_plan` | Generates fix instructions |
| Direct Reply (PASS) | DirectReply | N/A | Success path terminal |
| Direct Reply (FAIL) | DirectReply | N/A | Failure path terminal |

**Validation Gate Configuration:**
- **Max Retries:** 3
- **Paths:** PASS (tests pass) | FIX (tests fail, retries < 3) | FAIL (tests fail, retries ≥ 3)

---

## Test Cases

### TC-5.1: Pass on First Attempt (No Loop)

**Objective:** Verify workflow exits immediately when code passes validation on first attempt

**Input:**
```
Write a Python function that adds two numbers together.
```

**Expected Execution Flow:**

1. **Agent.Generate Executes (Attempt 1):**
   - Generates code:
     ```python
     def add_numbers(a, b):
         return a + b
     ```
   - Updates state:
     ```json
     {
       "loop.code": "[generated code]",
       "loop.attempt": 1
     }
     ```

2. **Agent.Validate Executes:**
   - Runs automated tests on generated code
   - Test cases:
     ```python
     assert add_numbers(2, 3) == 5
     assert add_numbers(-1, 1) == 0
     assert add_numbers(0, 0) == 0
     ```
   - All tests pass ✅
   - Updates state:
     ```json
     {
       "loop.validation_result": "PASS",
       "loop.errors": []
     }
     ```

3. **Validation Gate:**
   - Reads `loop.validation_result = "PASS"`
   - Decision: **PASS** → Route to success Direct Reply

4. **Direct Reply (PASS) Returns:**
   - Displays:
     ```
     ✅ Code validation successful!

     Generated code passed all tests on first attempt.

     Attempts: 1
     Status: PASS
     ```

**Validation Checklist:**

- [ ] **Generate Execution:**
  - Generated valid Python code
  - State contains `loop.code`
  - State contains `loop.attempt=1`

- [ ] **Validate Execution:**
  - Tests executed on generated code
  - All tests passed
  - State contains `loop.validation_result="PASS"`
  - State contains `loop.errors=[]` (empty)

- [ ] **Gate Decision:**
  - Read `loop.validation_result="PASS"`
  - Routed to PASS path
  - **CRITICAL:** Did NOT route to FIX or FAIL path

- [ ] **Loop Behavior:**
  - Agent.FixPlan did NOT execute
  - Agent.Generate did NOT execute again
  - Loop exited immediately on first success

- [ ] **Success Response:**
  - Direct Reply (PASS) displayed
  - Response indicates attempt count (1)
  - Response confirms all tests passed

**Success Criteria:**
- Code passed validation on first attempt
- Loop exited immediately (no retries)
- Attempt count = 1
- No fix plan generated

---

### TC-5.2: Retry Loop with Successful Fix

**Objective:** Verify loop retries with fixes until code passes validation

**Input:**
```
Write a Python function that reverses a string.
```

**Expected Execution Flow:**

1. **Agent.Generate (Attempt 1):**
   - Generates code with bug:
     ```python
     def reverse_string(s):
         return s[::-1]  # BUG: doesn't handle None input
     ```
   - Updates state: `loop.code`, `loop.attempt=1`

2. **Agent.Validate:**
   - Runs tests
   - Test failures:
     ```
     ✅ PASS: reverse_string("hello") == "olleh"
     ❌ FAIL: reverse_string(None) raises TypeError
     ```
   - Updates state:
     ```json
     {
       "loop.validation_result": "FIX",
       "loop.errors": ["TypeError: 'NoneType' object is not subscriptable"]
     }
     ```

3. **Validation Gate (After Attempt 1):**
   - Reads `loop.validation_result="FIX"`
   - Reads `loop.attempt=1 < 3`
   - Decision: **FIX** → Route to FixPlan

4. **Agent.FixPlan:**
   - Analyzes error: "TypeError with None input"
   - Generates fix plan:
     ```
     Add None check at start of function:
     if s is None:
         return ""
     ```
   - Updates state: `loop.fix_plan`

5. **Agent.Generate (Attempt 2):**
   - Reads `loop.fix_plan` from state
   - Generates fixed code:
     ```python
     def reverse_string(s):
         if s is None:
             return ""
         return s[::-1]
     ```
   - Updates state: `loop.code`, `loop.attempt=2`

6. **Agent.Validate:**
   - Runs tests again
   - All tests pass ✅
   - Updates state: `loop.validation_result="PASS"`

7. **Validation Gate (After Attempt 2):**
   - Reads `loop.validation_result="PASS"`
   - Decision: **PASS** → Route to success Direct Reply

8. **Direct Reply (PASS) Returns:**
   - Displays:
     ```
     ✅ Code validation successful!

     Initial attempt failed but was fixed automatically.

     Attempts: 2
     Fixes applied: 1
     Status: PASS
     ```

**Validation Checklist:**

- [ ] **Attempt 1:**
  - Generated code with bug
  - Validation failed with specific error
  - State contains `loop.validation_result="FIX"`
  - State contains `loop.errors` with error details

- [ ] **Gate Decision (After Attempt 1):**
  - Read `loop.validation_result="FIX"`
  - Read `loop.attempt=1 < 3`
  - Routed to FIX path (NOT PASS or FAIL)

- [ ] **FixPlan Execution:**
  - Analyzed error from `loop.errors`
  - Generated specific fix instructions
  - State contains `loop.fix_plan` with fix details

- [ ] **Loop-Back:**
  - Agent.Generate executed again (Attempt 2)
  - Generate read `loop.fix_plan` from state
  - Generated code incorporated fix

- [ ] **Attempt 2:**
  - Code passed validation
  - State contains `loop.validation_result="PASS"`
  - State contains `loop.attempt=2`

- [ ] **Loop Exit:**
  - Gate routed to PASS after Attempt 2
  - Loop exited successfully
  - No Attempt 3 occurred

- [ ] **Success Response:**
  - Indicates 2 attempts total
  - Confirms fix was applied
  - Status = PASS

**Success Criteria:**
- Initial code failed validation
- Fix plan generated with specific instructions
- Fixed code passed validation
- Loop executed exactly 2 times
- Final status = PASS

---

### TC-5.3: Max Retries Reached - FAIL Path

**Objective:** Verify loop terminates at max retries (3) and routes to FAIL path

**Input:**
```
Write a Python function that parses and validates ISO 8601 datetime strings.
```

**Expected Execution Flow:**

1. **Attempt 1:**
   - Generate → Validate → Error: "Incorrect timezone parsing"
   - Gate → FIX (retry 1 < 3)
   - FixPlan → Generate (loop back)

2. **Attempt 2:**
   - Generate (with fix plan) → Validate → Error: "Missing edge case: leap seconds"
   - Gate → FIX (retry 2 < 3)
   - FixPlan → Generate (loop back)

3. **Attempt 3:**
   - Generate (with fix plan) → Validate → Error: "Still failing: UTC offset validation"
   - Gate → Reads `loop.attempt=3 ≥ 3` → **FAIL**

4. **Direct Reply (FAIL) Returns:**
   - Displays:
     ```
     ❌ Code validation failed after 3 attempts.

     Unable to generate passing code after maximum retry limit.

     Attempts: 3
     Status: FAIL
     Last error: UTC offset validation failed

     Manual intervention required.
     ```

**Validation Checklist:**

- [ ] **Attempt Tracking:**
  - Attempt 1: validation_result="FIX", attempt=1
  - Attempt 2: validation_result="FIX", attempt=2
  - Attempt 3: validation_result="FIX", attempt=3

- [ ] **Gate Decisions:**
  - After Attempt 1: FIX (1 < 3) → loop back
  - After Attempt 2: FIX (2 < 3) → loop back
  - After Attempt 3: FAIL (3 ≥ 3) → exit to FAIL path

- [ ] **Max Retries Enforcement:**
  - Exactly 3 attempts executed
  - **CRITICAL:** No 4th attempt occurred
  - Loop terminated at max retries

- [ ] **FAIL Path Execution:**
  - Gate routed to FAIL (NOT FIX)
  - Direct Reply (FAIL) displayed
  - Agent.FixPlan did NOT execute after Attempt 3

- [ ] **FAIL Response:**
  - Indicates 3 attempts
  - Status = FAIL
  - Includes last error details
  - Suggests manual intervention

- [ ] **No Infinite Loop:**
  - Loop terminated after 3 attempts
  - Workflow completed (didn't hang)

**Success Criteria:**
- Loop executed exactly 3 times (max enforced)
- Gate correctly routed to FAIL after 3 attempts
- No 4th attempt occurred
- FAIL Direct Reply displayed with helpful error message

**Max Retries Safety Validation:**
```
Expected Behavior:

Attempt 1: validation="FIX", attempts=1 < 3 → FIX path (loop back) ✅
Attempt 2: validation="FIX", attempts=2 < 3 → FIX path (loop back) ✅
Attempt 3: validation="FIX", attempts=3 ≥ 3 → FAIL path (exit) ✅
[STOP - No Attempt 4]

✅ PASS: Loop terminated at 3 attempts
❌ FAIL: Attempt 4 executed (infinite loop bug!)
```

---

## Common Issues & Debugging

### Issue 1: Infinite Loop (Attempts Continue Beyond 3)

**Symptoms:**
- Generate executes 4, 5, or more times
- Loop never exits to FAIL path
- Workflow hangs

**Debugging Steps:**
1. Verify Validation Gate has FAIL condition:
   - Should check `loop.attempt ≥ 3`
   - Should have separate output anchor for FAIL
   - FAIL should route to Direct Reply (FAIL), not FixPlan

2. Check attempt counter increment:
   - Agent.Generate should increment `loop.attempt` by 1 each time
   - Verify state update: `loop.attempt = {{ loop.attempt + 1 }}`

3. Verify edge connections:
   ```json
   // Gate should have 3 outputs:
   { "source": "validation_gate", "sourceHandle": "output-0", "target": "directReply_pass" },  // PASS
   { "source": "validation_gate", "sourceHandle": "output-1", "target": "agent_fixplan" },     // FIX
   { "source": "validation_gate", "sourceHandle": "output-2", "target": "directReply_fail" }   // FAIL
   ```

**Solution:**
- Add FAIL condition checking `loop.attempt ≥ 3`
- Ensure Generate increments attempt counter
- Verify FAIL routes to Direct Reply (FAIL), NOT FixPlan

---

### Issue 2: PASS Path Displays Even When Tests Fail

**Symptoms:**
- Tests fail but PASS Direct Reply displays
- No retry loop occurs
- Validation errors ignored

**Debugging Steps:**
1. Check Validate agent sets `loop.validation_result`:
   - Should be "PASS" when tests pass
   - Should be "FIX" when tests fail (and retries < 3)
   - Should be "FAIL" when tests fail (and retries ≥ 3)

2. Verify Validation Gate reads correct state variable:
   - Should read `{{ loop.validation_result }}`
   - Variable name must match Validate state update

3. Check gate scenario order:
   - Scenario 0: PASS (validation_result="PASS")
   - Scenario 1: FIX (validation_result="FIX", attempts < 3)
   - Scenario 2: FAIL (validation_result="FIX", attempts ≥ 3)

**Solution:**
- Configure Validate to set `loop.validation_result` correctly
- Verify state variable names match
- Check gate conditions for each path

---

### Issue 3: FixPlan Not Applied to Next Attempt

**Symptoms:**
- Fix plan generated but not used
- Same bug appears in Attempt 2
- Code doesn't improve between attempts

**Debugging Steps:**
1. Verify Agent.FixPlan updates state:
   - Should set `loop.fix_plan` with fix instructions
   - Fix plan should be specific and actionable

2. Check Agent.Generate reads fix plan:
   - Should access `{{ loop.fix_plan }}` from state
   - Prompt should instruct to apply fix plan

3. Verify loop-back edge:
   - FixPlan should connect to Generate (NOT Validate)
   - Check edge: `agent_fixplan → agent_generate`

**Solution:**
- Configure FixPlan to update `loop.fix_plan` state
- Ensure Generate reads fix plan from state
- Verify loop-back edge connects FixPlan → Generate

---

### Issue 4: FAIL Path Executes Immediately (No Retries)

**Symptoms:**
- First attempt fails and immediately routes to FAIL
- No retry loop occurs
- Attempt count = 1 but FAIL path used

**Debugging Steps:**
1. Check Validation Gate FIX condition:
   - Should route to FIX when `validation_result="FIX" AND attempts < 3`
   - Should NOT route to FAIL when attempts = 1 or 2

2. Verify attempt counter initialization:
   - Should start at 1 (not 3)
   - Check Agent.Generate initial state update

3. Check gate scenario order:
   - FIX scenario should come before FAIL scenario
   - Gate should check attempts < 3 condition

**Solution:**
- Set FIX condition to check `attempts < 3`
- Initialize attempt counter to 1
- Ensure FIX scenario has higher priority than FAIL

---

### Issue 5: Validation Agent Doesn't Run Tests

**Symptoms:**
- Validation always returns PASS
- No actual test execution
- No error details in `loop.errors`

**Debugging Steps:**
1. Verify Validate agent has test execution capability:
   - Should use code execution tool or API
   - Should run actual tests on generated code

2. Check Validate prompt includes test cases:
   - Prompt should specify test cases to run
   - Should capture test results (pass/fail)

3. Verify Validate updates state with results:
   - Should set `loop.validation_result` based on test results
   - Should set `loop.errors` with failure details

**Solution:**
- Configure Validate to execute actual tests
- Add test cases to Validate prompt
- Ensure state updates reflect test results accurately

---

## Test Execution Report

**Test Date:** `[YYYY-MM-DD]`
**Flowise Version:** `[version]`
**Pattern Version:** `1.0`

### Test Results Summary

| Test Case | Status | Attempts | Final Result | Notes |
|-----------|--------|----------|--------------|-------|
| TC-5.1: Pass First Attempt | ⬜ Pass / ⬜ Fail | `[1]` | `[PASS]` | |
| TC-5.2: Retry with Fix | ⬜ Pass / ⬜ Fail | `[2]` | `[PASS]` | |
| TC-5.3: Max Retries | ⬜ Pass / ⬜ Fail | `[3]` | `[FAIL]` | |

### Configuration Details

- **Anthropic Model:** `claude-sonnet-4-5-20250929`
- **Temperature:** `0.2`
- **Max Retries:** `3`
- **Streaming:** `Enabled`
- **Memory Enabled:** `Yes`

### Retry Loop Performance Metrics

| Test Case | Attempt 1 | Attempt 2 | Attempt 3 | Exit Path |
|-----------|-----------|-----------|-----------|-----------|
| TC-5.1 | PASS | N/A | N/A | PASS (early) |
| TC-5.2 | FIX | PASS | N/A | PASS |
| TC-5.3 | FIX | FIX | FIX | FAIL (max retries) |

### Issues Encountered

`[List any issues encountered during testing]`

### Additional Notes

`[Any additional observations or recommendations]`

---

## Pattern Validation

This pattern has been validated against `validate_workflow.py` with the following checks:

✅ **Structural Validation:**
- Proper JSON structure
- All required node and edge fields present

✅ **Node Type Validation:**
- Start node present
- 1 Generate agent
- 1 Validate agent
- 1 Validation Gate (ConditionAgent) with 3 output anchors
- 1 FixPlan agent
- 2 DirectReply nodes (PASS and FAIL)
- Sticky Notes for documentation

✅ **Loop Logic:**
- Loop-back edge exists (FixPlan → Generate)
- Max retries enforced (≤ 3)
- 3-path gate (PASS/FIX/FAIL)
- FAIL path prevents returning broken code

✅ **State Management:**
- All agents have memory enabled
- Attempt counter increments correctly
- Validation result tracked
- Fix plan persisted for next iteration

✅ **Safety Validation:**
- No infinite loop possible (max 3 attempts enforced)
- FAIL path exits gracefully
- Broken code NOT returned to user

✅ **Validation Gate Configuration:**
- 3 scenarios:
  - PASS: validation_result="PASS"
  - FIX: validation_result="FIX" AND attempts < 3
  - FAIL: attempts ≥ 3

---

## Additional Resources

- **Pattern Library:** [Context Foundry AFv2 Patterns](https://github.com/context-foundry/context-foundry/tree/main/extensions/flowise/templates/afv2-patterns)
- **Pattern #5 Repository:** https://github.com/snedea/afv2-pattern-05-looping
- **Flowise Documentation:** https://docs.flowiseai.com/
- **AgentFlow v2 Specification:** [AFv2 Docs](https://docs.flowiseai.com/agentflows)

---

🤖 Built with Context Foundry
