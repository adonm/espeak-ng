# Technical Debt Registry

**Status**: Initial Assessment  
**Last Updated**: 2026-02-02

---

## FIXME/TODO Inventory

### Critical Priority

#### 1. Phoneme Classification Issues
**File**: `src/libespeak-ng/phoneme.c`

| Line | Code | Issue | Impact |
|------|------|-------|--------|
| 48 | `case afr: // FIXME: eSpeak treats 'afr' as 'stp'` | Africates misclassified as stops | Incorrect phoneme duration |
| 52 | `case apr: // FIXME: eSpeak is using this for [h], with 'liquid' used for [l] and [r]` | Approximant vs liquid confusion | Wrong phoneme generation |
| 55 | `case flp: // FIXME: Why is eSpeak using a vstop (vcd + stp) for this?` | Flap incorrectly coded as voiced stop | Incorrect articulation |
| 58 | `case trl: // FIXME: 'trill' should be the type; 'liquid' should be a flag` | Trill and liquid conflated | Phonetic inaccuracy |
| 226 | `case elg: // FIXME: Should be longer than 'lng'` | Elongated duration wrong | Timing issues |

**Action Required**: 
- Review IPA phoneme definitions
- Update phoneme type system
- Regenerate phoneme data files
- Add tests for affected phonemes

---

#### 2. Multi-Process MBROLA Support
**File**: `src/libespeak-ng/mbrowrap.c:22`

```c
/* FIXME: we should be able to run several mbrola processes,
```

**Impact**: Cannot use multiple MBROLA voices concurrently

**Action Required**:
- Refactor mbrowrap to support multiple instances
- Add process pool management
- Implement proper resource cleanup

---

### High Priority

#### 3. Unsafe String Operations

**File**: `src/libespeak-ng/soundicon.c:92`
```c
strcpy(fname_temp, "/tmp/espeakXXXXXX");  // SECURITY ISSUE
```

**Risk**: Buffer overflow, path traversal

**Fix**: 
```c
strncpy(fname_temp, "/tmp/espeakXXXXXX", sizeof(fname_temp) - 1);
fname_temp[sizeof(fname_temp) - 1] = '\0';
// Better: use mkstemp() for secure temp file
```

---

**File**: `src/libespeak-ng/compiledict.c`
```c
sprintf(buf, "...");  // Multiple instances
```

**Risk**: Buffer overflow in dictionary compilation

**Fix**: Replace all sprintf with snprintf

---

#### 4. Dictionary Compilation Bug
**File**: `src/libespeak-ng/compiledict.c:316`

```c
// TODO write the string backwards if in RULE_PRE
```

**Impact**: Incorrect rule compilation for prefix rules

---

### Medium Priority

#### 5. Hardcoded Paths

**File**: `src/libespeak-ng/speech.c`
```c
RegOpenKeyExA(HKEY_LOCAL_MACHINE, "Software\\eSpeak NG", 0, KEY_READ, &RegKey);
```

**Issue**: Hardcoded registry path on Windows

---

#### 6. Magic Numbers

**Files**: Multiple

Examples:
- `N_PHONEME_LIST` - No documentation on why this size
- `N_WORD_BYTES` - Arbitrary limit on word length
- Buffer sizes throughout codebase

**Action**: Document all magic numbers or make configurable

---

## Global State Variables

### Configuration Globals (translate.c)
```c
int option_tone_flags = 0;
int option_phonemes = 0;
int option_phoneme_events = 0;
int option_endpause = 0;
int option_capitals = 0;
int option_punctuation = 0;
int option_sayas = 0;
int option_ssml = 0;
int option_phoneme_input = 0;
int option_wordgap = 0;
int option_linelength = 0;
```

**Impact**: Cannot have multiple concurrent synthesis contexts
**Solution**: Move to espeak_context_t struct

---

### Translator State (translate.c)
```c
Translator *translator = NULL;
Translator *translator2 = NULL;
Translator *translator3 = NULL;
```

**Impact**: Single global translator state
**Solution**: Pass translator context through API

---

### Output State
```c
FILE *f_trans = NULL;  // Phoneme output file
int n_ph_list2;
PHONEME_LIST2 ph_list2[N_PHONEME_LIST];
```

---

## File Size Issues

### Files Requiring Splitting

| File | Lines | Issue | Refactor Target |
|------|-------|-------|----------------|
| `translate.c` | 1,807 | Too many responsibilities | Split into 3-4 modules |
| `dictionary.c` | 3,053 | Lookup + compilation logic | Separate concerns |
| `synthesize.c` | 1,607 | Multiple synthesizers | Plugin architecture |
| `voices.c` | 1,449 | Loading + management | Split loader/manager |
| `compiledata.c` | 2,794 | Complex compilation | Break into stages |
| `tr_languages.c` | 1,704 | Language-specific code | Extract per-language modules |

---

## Code Quality Issues

### 1. Tight Coupling

**Issue**: Modules include each other's headers excessively

**Example**:
```c
// synthesize.c includes:
#include "dictionary.h"
#include "translate.h"
#include "voice.h"
#include "wavegen.h"
// ... etc
```

**Solution**: Define clear module interfaces

---

### 2. Mixed Concerns

**Example**: `translate.c` handles:
- Text segmentation
- Word translation
- Language detection
- Phoneme generation
- SSML parsing

**Solution**: Separate into focused modules

---

### 3. Lack of Error Handling

**Pattern**: Many functions return void or don't check return values

**Example**:
```c
// No error checking
strcpy(dest, src);
memcpy(buffer, data, size);
```

**Solution**: Add error propagation throughout

---

## Dependencies

### Build System
- **Dual build systems** (autotools + CMake) - Confusing for contributors
- **Action**: Remove autotools completely

### External Dependencies
| Dependency | Status | Action |
|------------|--------|--------|
| pcaudiolib | Optional | Keep, improve integration |
| sonic | Optional | Keep, improve integration |
| ronn | Build-only | Optional, remove if problematic |
| kramdown | Docs-only | Keep for documentation |

### Internal Dependencies
- **ucd-tools**: Unicode database - Keep, update to latest Unicode
- **MBROLA**: Legacy integration - Deprecate? Or modernize?

---

## Performance Issues

### 1. Linear Search in Dictionary
**File**: `dictionary.c`

**Issue**: Dictionary lookups use linear search

**Solution**: Implement hash table or trie

---

### 2. Memory Allocation
**Pattern**: Frequent small allocations

**Issue**: Fragmentation, overhead

**Solution**: Use memory pools, arena allocators

---

### 3. No Streaming Support
**Issue**: Must buffer entire text before synthesis

**Solution**: Implement incremental processing

---

## Documentation Debt

### Missing Documentation
- [ ] Architecture overview
- [ ] Module interaction diagrams
- [ ] API usage examples
- [ ] Performance characteristics
- [ ] Build system guide (CMake)
- [ ] Testing guide

### Outdated Documentation
- [ ] Autotools build instructions
- [ ] Some man pages
- [ ] Language addition guide (partially)

---

## Testing Debt

### Missing Test Coverage
- [ ] Phoneme accuracy for most languages
- [ ] Dictionary compilation
- [ ] Voice loading
- [ ] SSML parsing edge cases
- [ ] Performance benchmarks
- [ ] Memory safety

### Testing Infrastructure
- [ ] No fuzzing for most components
- [ ] No performance regression testing
- [ ] No audio quality testing

---

## Next Actions

### Immediate (This Week)
1. Fix security issues (strcpy, sprintf)
2. Resolve critical FIXME comments
3. Document global state

### Short Term (Next Month)
1. Refactor translate.c
2. Create module boundaries
3. Add comprehensive error handling

### Medium Term (Phase 3)
1. Complete modularization
2. Implement new API
3. Remove autotools

---

## Risk Assessment

### High Risk
- **Breaking changes** may affect screen reader users
- **Large refactor** could introduce regressions
- **New maintainers** need training

### Mitigation
- Comprehensive test suite before changes
- Gradual migration with compatibility layer
- Community engagement and testing

---

**Note**: Update this registry as issues are resolved or new ones discovered.
