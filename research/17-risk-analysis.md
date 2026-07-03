# 17 — Risk Analysis: Rebuilding Quantbit from Scratch

---

## Risk Matrix

| # | Risk | Probability | Impact | Level | Mitigation |
|----|------|-------------|--------|-------|------------|
| R1 | Data pipeline failure (Yahoo API changes) | Medium | Critical | **HIGH** | Multiple data sources, graceful degradation |
| R2 | Business logic misinterpretation | High | High | **HIGH** | Cross-reference with old backtest results |
| R3 | Scope creep — too many features at once | High | Medium | **HIGH** | Strict MVP definition, freeze scope after Sprint 3 |
| R4 | Team loses momentum (long rebuild) | Medium | High | **HIGH** | Working software setiap sprint, demo setiap minggu |
| R5 | AI provider geo-blocking | Medium | Medium | **MEDIUM** | OpenRouter sebagai primary (tidak geo-block), fallback lokal |
| R6 | Backward compatibility with old data | Low | Medium | **MEDIUM** | Data migration scripts, parallel run validation |
| R7 | User migration (v1 → v2) | Medium | Medium | **MEDIUM** | Grace period, data export, clear communication |
| R8 | Performance regression in engine | Low | High | **MEDIUM** | Benchmark suite from day 1, compare with v1 results |
| R9 | Security incident during rebuild | Low | Critical | **MEDIUM** | Security review setiap sprint, penetration test sebelum launch |
| R10 | Dependency abandonment (D1, Hono) | Low | Medium | **LOW** | Framework-agnostic core, abstraction layers |
| R11 | Fundamental scoring inaccuracy | Medium | High | **HIGH** | Validate against known market data, statistical analysis |
| R12 | Regulatory compliance (financial data) | Low | Critical | **MEDIUM** | Consult legal, data retention policy, disclaimers |

---

## Detailed Risk Analysis

### R1: Yahoo Finance API Changes or Deprecation
**Probability**: Medium — Yahoo Finance API is undocumented, has changed multiple times
**Impact**: Critical — entire data pipeline depends on Yahoo Finance (historical prices, IDX80 scan)

**Current State**: The old project fetches from `query1.finance.yahoo.com` which is a reverse-engineered endpoint. No official API key, no contract, no SLA.

**Mitigation**:
- Abstract data fetching behind `DataProvider` interface
- Implement 2+ providers (Yahoo + IDX API + GoAPI)
- Cache aggressively so pipeline works with stale data during outages
- Alert when data freshness exceeds threshold

### R2: Business Logic Misinterpretation
**Probability**: High — 36+ business rules extracted, some subtle
**Impact**: High — wrong numbers erode user trust

**Example Risks**:
- BPS factor weights misread (0.30 vs 0.25)
- Crash sensitivity formula wrong (`-crashSensitivity` vs `<` vs `<=`)
- Dividend accrual date wrong (June 15 vs June 1)
- Fee calculation wrong (buy 0.15% vs sell 0.25%)

**Mitigation**:
- Every extracted business rule must be verified against the actual code
- Run old backtest data through new engine → compare results
- Unit test each formula independently
- Integration test with known inputs/outputs

### R3: Scope Creep
**Probability**: High — "since we're rebuilding, let's also add X"
**Impact**: Medium — delays launch, increases complexity

**Current State**: Old project has 39+ features, many experimental or unused

**Mitigation**:
- **Hard MVP boundary**: Market → Portfolio → Backtest → Basic AI
- Everything else (MCP, adaptive weights, multiple AI providers, proactive alerts) = post-MVP
- Sprint 0 document freeze: feature list locked, changes reviewed by architect

### R4: Team Loses Momentum
**Probability**: Medium — 10+ sprint rebuild is long
**Impact**: High — abandoned rebuild is worst outcome

**Mitigation**:
- Every sprint must ship working software
- Prioritize visible progress (UI first, engine second)
- Demo-day setiap sprint dengan stakeholder
- Celebrate milestones (first backtest run, first AI response)

### R5: AI Provider Geo-Blocking
**Probability**: Medium — known issue in old project (aiChatHandler.ts comments)
**Impact**: Medium — AI features don't work for some users

**Current State**: "NOTE: Direct API calls (Groq, Google) are geo-blocked from many Cloudflare Pages edge locations."

**Mitigation**:
- OpenRouter as primary provider (routes through their pool, not geo-blocked)
- Local fallback (dev mock) for when no provider works
- Clearly documented provider status in UI

### R6: Backward Compatibility with Old Data
**Probability**: Low — we're rebuilding from scratch
**Impact**: Medium — user-created data (portfolio, watchlists) may not migrate

**Mitigation**:
- Migration script for D1 data
- CSV export option from old system
- Parallel run period where both v1 and v2 are available

### R11: Fundamental Scoring Inaccuracy
**Probability**: Medium
**Impact**: High — wrong stock ranking destroys usefulness

**Current State**: Old project uses synthetic fundamentals for most stocks (hardcoded for ~10 tickers, generated for rest). Quality/growth/value/momentum computed from price data alone in `runIdx80Scan()`.

**New Approach**: Use IDX API for real fundamental data (ADR-006). Better data = better scores.

**Mitigation**:
- Validate computed scores against known market data
- Statistical analysis: distribution should be roughly normal
- Pin to known results (top 10 tickers by score should make sense)

---

## Risk Response Plan

### For HIGH Risks

| Risk | Trigger | Response |
|------|---------|----------|
| R1 | Yahoo Finance endpoint changes | Switch to backup provider, degrade gracefully |
| R2 | Backtest results differ from v1 | Investigate formula, compare line by line |
| R3 | Stakeholder requests new feature | Document as post-MVP, do not add to current sprint |
| R4 | 2 sprints without demo-able progress | Reduce scope, ship smaller increments |
| R11 | Scores don't match market reality | Re-validate formulas, check data quality |

### For MEDIUM Risks

| Risk | Trigger | Response |
|------|---------|----------|
| R5 | AI responses fail in production | Switch to OpenRouter-only, document limitations |
| R6 | User data migration fails | Manual export/import tool |
| R7 | Users don't migrate | Grace period, offer both versions temporarily |
| R8 | Engine is slower than v1 | Profile, optimize, consider Web Workers |
| R9 | Security audit finds issues | Dedicated security sprint |
| R12 | Legal questions | Engage legal counsel, add disclaimers |

---

## Risk Burndown by Sprint

| Sprint | Active Risks | Mitigation |
|--------|--------------|------------|
| Sprint 0 | R3 (scope creep) | Freeze feature list |
| Sprint 1 | R1 (Yahoo API), R6 (data migration) | Test data fetching, write migration |
| Sprint 2 | R2 (formula misinterpretation) | Cross-validate with v1 results |
| Sprint 3 | R8 (performance) | Benchmark against v1 |
| Sprint 4 | R9 (security) | Security review |
| Sprint 5 | R4 (momentum) | Demo working UI |
| Sprint 6 | R2 (backtest accuracy) | Integration tests |
| Sprint 7 | R5 (geo-blocking) | Test AI in production |
| Sprint 8 | R8, R9 | Performance + security |
| Sprint 9 | R7 (user migration) | Launch + migration guide |

---

## Worst Case Scenario

**What happens if all risks materialize?**
1. Yahoo Finance blocks access → data pipeline fails
2. Formula misinterpretation → backtest results wrong
3. Scope creep → project never finishes
4. Team loses momentum → rebuild abandoned
5. AI geo-blocked → AI features broken
6. Migration fails → users lose data
7. Performance regression → engine unusable
8. Security incident → data breach

**Recovery Plan**:
- Fall back to static data files (like v1)
- Pause new features, fix formulas
- Reduce scope to absolute minimum (market overview + portfolio only)
- Use old system as backup while fixing new one
- Implement emergency data purge if breach occurs

---

## Risk Owner Assignment

| Risk | Owner | Review Frequency |
|------|-------|------------------|
| R1 — Data pipeline | Data Engineer | Weekly |
| R2 — Business logic | Quant Developer | Each formula change |
| R3 — Scope creep | Architect | Sprint planning |
| R4 — Momentum | Tech Lead | Daily standup |
| R5 — AI geo-blocking | AI Developer | Sprint review |
| R6 — Backward compat | Data Engineer | Migration sprint |
| R7 — User migration | Product Owner | Pre-launch |
| R8 — Performance | Quant Developer | Each sprint |
| R9 — Security | Architect | Each sprint |
| R11 — Scoring | Quant Developer | Each formula change |
| R12 — Compliance | Product Owner | Pre-launch |

---

## Go/No-Go Criteria

**Before starting Sprint 1**, verify:
- [x] Data pipeline produces correct data (compare with v1)
- [x] Engine produces matching results (sampled backtest)
- [x] MVP feature list is frozen
- [x] Team is committed for 10+ sprints
- [x] Security baseline is established

**Before MVP launch** (Sprint 9), verify:
- [ ] All P0/P1 security issues fixed
- [ ] Backtest accuracy within 1% of v1
- [ ] Data pipeline has been running for 2+ weeks without failure
- [ ] AI chat works for target user base
- [ ] User migration tested with real data
- [ ] Performance meets or exceeds v1
