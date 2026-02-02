# Quick Reference Guide

**Project**: eSpeak-NG Modernization  
**Branch**: `modernization/technical-debt-plan`  

---

## Document Map

| Document | Purpose | When to Read |
|----------|---------|--------------|
| `ROADMAP.md` | Complete modernization plan | Start here |
| `TECH_DEBT.md` | Technical debt registry | During Phase 1-2 |
| `AI_TESTING.md` | Testing framework design | During Phase 5 |
| `ARCHITECTURE.md` | System architecture (TODO) | During Phase 1 |
| `API_PROPOSAL.md` | New API specification (TODO) | During Phase 3 |

---

## Phase Quick Reference

### Phase 1: Analysis (Weeks 1-2)
```bash
# Tasks
- Document current architecture
- Map global state
- Catalog dependencies

# Deliverables
docs/modernization/ARCHITECTURE.md
docs/modernization/DATA_FLOW.md
```

### Phase 2: Critical Fixes (Weeks 3-4)
```bash
# Tasks
- Fix security issues
- Resolve FIXME comments
- Reduce global state

# Key Files to Edit
src/libespeak-ng/phoneme.c      # Fix phoneme issues
src/libespeak-ng/soundicon.c    # Fix strcpy
src/libespeak-ng/compiledict.c  # Fix sprintf
```

### Phase 3: Refactoring (Weeks 5-12)
```bash
# Tasks
- Split large files
- Create modules
- Implement new API

# New Structure
src/core/
├── api/
├── text-processing/
├── phoneme/
├── synthesis/
├── audio/
└── voice/
```

### Phase 4: Build System (Weeks 13-14)
```bash
# Tasks
- Remove autotools
- Complete CMake migration
- Add vcpkg/conan support

# Commands
rm configure.ac Makefile.am
vim CMakeLists.txt  # Add exports
```

### Phase 5: AI Testing (Weeks 15-18)
```bash
# Tasks
- Build testing framework
- Integrate AI validation
- Set up CI/CD

# Run Tests
cd tests/ai_framework
python -m pytest validators/
```

### Phase 6: Language Foundation (Weeks 19-20)
```bash
# Tasks
- Add metadata to language files
- Set up quality metrics
- Create documentation

# Example Language File
name French
language fr
maintainer contact@example.com
quality verified
last_reviewed 2024-01-15
```

---

## Key Metrics

### Success Criteria
- **FIXMEs**: 0 unresolved
- **Test Coverage**: >80%
- **Lines/File**: <500
- **Languages Tested**: All 100+
- **Latency**: <10ms first phoneme

### Current State
- FIXMEs: 17
- Test Coverage: Low
- Lines/File: Up to 3,053
- Languages: 100+ (limited testing)
- Latency: Unknown baseline

---

## Common Commands

### Build
```bash
# CMake (preferred)
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)

# With sanitizers
cmake .. -DUSE_ASAN=ON -DUSE_UBSAN=ON

# Test
make check
```

### Debug
```bash
# Compile with debug info
espeak-ng --compile-debug=en

# Trace rules
espeak-ng -ven -X "Test text"

# Get phonemes
espeak-ng -ven -x "Test text"
```

---

## Decision Points

### API Compatibility
- **Option A**: Breaking changes (2.0.0) ← **Chosen**
- **Option B**: Compatibility shim

### AI Testing Provider
- **Option A**: OpenAI API (better quality)
- **Option B**: Local LLM (lower cost)
- **Option C**: Hybrid approach ← **Recommended**

### Refactoring Strategy
- **Option A**: Big-bang (faster, riskier)
- **Option B**: Incremental (slower, safer) ← **Chosen**

---

## Contact & Resources

- **Mailing List**: https://groups.io/g/espeak-ng
- **Issues**: https://github.com/espeak-ng/espeak-ng/issues
- **Wiki**: https://github.com/espeak-ng/espeak-ng/wiki
- **Original Article**: https://stuff.interfree.ca/2026/01/05/ai-tts-for-screenreaders.html

---

## Status Legend

- **TODO**: Not started
- **WIP**: Work in progress
- **REVIEW**: Needs review
- **DONE**: Complete

---

## Branch Management

```bash
# Current branch
git branch
# modernization/technical-debt-plan

# Keep updated with master
git fetch origin
git rebase origin/master

# Before starting work
git checkout modernization/technical-debt-plan
git pull origin modernization/technical-debt-plan

# Create feature branches for work
git checkout -b feature/phase2-security-fixes
git checkout -b feature/phase3-modularization
```

---

**Last Updated**: 2026-02-02
