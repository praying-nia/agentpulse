# AgentPulse repository instructions

## Development log

- Read `DEVELOPMENT_LOG.md` before starting implementation work, then verify its claims against the relevant repositories.
- After completing and verifying a milestone, update `DEVELOPMENT_LOG.md` in the same change:
  - refresh the current status;
  - add a dated history entry with completed work, decisions, verification, commit state, and remaining work;
  - set exactly one concrete next target.
- Do not record exploration, plans, incomplete work, or unverified changes as completed.
- Describe Git state accurately. Do not claim that changes are committed or pushed unless repository state confirms it.
- Persistence remains undecided. Do not introduce or document a database as an architectural requirement without an explicit project decision.

## Complete milestone principle

- Build complete, usable software capabilities rather than demo-sized fragments. Every milestone must close its declared boundary with production behavior, failure handling, documentation, and proportionate verification.
- Completeness applies to the explicitly chosen scope and platform boundary. Reassess overall product progress before choosing the next target, and do not expand sideways into unrelated features merely to make one milestone appear larger.
