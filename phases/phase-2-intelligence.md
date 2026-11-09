# Phase 2: Intelligence

## Goal
Add the System 1 Router — task classification and capability-based model selection.

## Work Items

### 2.1 Task Classification
- [ ] Implement task classifier (complexity + domain + required capabilities)
- [ ] Training data: examples of tasks mapped to complexity levels
- [ ] Classification output feeds router

### 2.2 Capability-Based Selection
- [ ] Map tasks to required capabilities (tools? files? browser? multi-step?)
- [ ] Query Resource Registry for matching available resources
- [ ] Filter by quota and availability
- [ ] Select smallest capable resource

### 2.3 Auto-Escalation
- [ ] Verification Engine detects insufficient results
- [ ] Router retries with next-stronger resource
- [ ] Escalation log recorded in Hermes session

## Deliverable
- Router takes a raw task, classifies it, picks a resource, executes, verifies,
  and escalates if needed.

## Verification
- [ ] Simple classification → small model
- [ ] Complex planning → strong reasoning model
- [ ] Insufficient result → escalation to stronger resource
