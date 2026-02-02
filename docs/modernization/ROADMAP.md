# eSpeak-NG Modernization Roadmap

**Status**: Planning Phase  
**Branch**: `modernization/technical-debt-plan`  
**Last Updated**: 2026-02-02

---

## Overview

This roadmap addresses critical technical debt in espeak-ng while laying groundwork for improved language quality. The approach prioritizes:

1. **Technical Debt First**: Address 30-year-old architecture issues
2. **Breaking Changes OK**: Modernize without backward compatibility constraints  
3. **AI-Assisted Testing**: Leverage modern tools for quality assurance
4. **Foundation for Language Quality**: Prepare infrastructure before addressing language issues

---

## Current State

### Codebase Metrics
- **Total Lines**: ~32,000 lines of C/C++
- **Core Files**: 24 source files in `src/libespeak-ng/`
- **Languages Supported**: 100+
- **Last Major Architecture Update**: 1995 (RISC OS origins)

### Critical Issues
- **FIXME Comments**: 17 unresolved technical issues
- **Global State**: ~30 global variables for configuration
- **Security**: Unsafe string operations (strcpy, sprintf)
- **Build System**: Dual autotools/CMake (transitioning)
- **Maintainability**: Large files (>1000 lines), tight coupling

---

## Phase Breakdown

### Phase 1: Codebase Analysis & Documentation (Weeks 1-2)
**Goal**: Understand and document current architecture

**Deliverables**:
- [ ] `docs/modernization/ARCHITECTURE.md` - System architecture documentation
- [ ] `docs/modernization/TECH_DEBT.md` - Technical debt registry with priorities
- [ ] `docs/modernization/DATA_FLOW.md` - Text-to-speech pipeline visualization
- [ ] `docs/modernization/API_PROPOSAL.md` - New API design

**Key Activities**:
- Map global variables and dependencies
- Identify critical path for real-time performance
- Document module coupling
- Catalog unsafe functions

---

### Phase 2: Critical Fixes & Safety (Weeks 3-4)
**Goal**: Fix immediate security and stability issues

**High-Priority Fixes**:

#### Security & Safety
```c
// src/libespeak-ng/soundicon.c:92
strcpy(fname_temp, "/tmp/espeakXXXXXX");  // → strncpy

// src/libespeak-ng/compiledict.c
sprintf(buffer, "...");  // → snprintf

// src/libespeak-ng/dictionary.c
// Multiple buffer overflow risks
```

#### FIXME Resolution
| File | Line | Issue | Priority |
|------|------|-------|----------|
| `phoneme.c` | 48 | 'afr' misclassified as 'stp' | High |
| `phoneme.c` | 52 | 'apr' vs 'liquid' confusion | High |
| `phoneme.c` | 55 | vstop usage for flp | Medium |
| `phoneme.c` | 58 | trill/liquid flag separation | Medium |
| `phoneme.c` | 226 | 'elg' duration too short | Low |
| `mbrowrap.c` | 22 | Multi-process mbrola support | Low |

#### Global State Reduction
**Current Pattern**:
```c
// translate.c - 30+ global options
int option_ssml = 0;
int option_phonemes = 0;
int option_endpause = 0;
// ... etc
```

**Target Pattern**:
```c
typedef struct espeak_config {
    struct {
        bool enable_ssml;
        bool enable_phonemes;
        bool endpause;
    } flags;
    int speed;
    int pitch;
    int volume;
} espeak_config_t;
```

---

### Phase 3: Modular Refactoring (Weeks 5-12)
**Goal**: Restructure codebase into logical modules

#### Proposed Directory Structure
```
src/
├── core/
│   ├── api/              # Public API layer
│   │   ├── espeak_api.c
│   │   └── context.c
│   ├── text-processing/  # Text → Phonemes
│   │   ├── text_processor.c    # Main orchestration
│   │   ├── clause_segmenter.c  # Text segmentation
│   │   ├── word_translator.c   # Word-level translation
│   │   ├── dictionary_lookup.c # Dictionary operations
│   │   ├── translate.c         # (refactored)
│   │   ├── translateword.c
│   │   └── dictionary.c
│   ├── phoneme/          # Phoneme operations
│   │   ├── phoneme.c
│   │   ├── phonemelist.c
│   │   └── phoneme_renderer.c
│   ├── synthesis/        # Synthesis engine
│   │   ├── synthesize.c
│   │   ├── synthdata.c
│   │   └── setlengths.c
│   ├── audio/            # Audio generation
│   │   ├── audio_pipeline.c
│   │   ├── wavegen.c
│   │   ├── klatt.c
│   │   └── sPlayer.c
│   └── voice/            # Voice management
│       ├── voices.c
│       ├── intonation.c
│       └── voice_loader.c
├── compat/               # Platform compatibility
├── data/                 # Compiled data handling
└── utils/                # Shared utilities
```

#### Refactoring Strategy

**Step 1: Split translate.c (1807 lines)**
- Extract clause segmentation logic
- Extract word translation logic
- Create clear interfaces between stages

**Step 2: Extract Dictionary Module**
- Separate dictionary loading from lookup
- Create cache layer for frequent lookups
- Add dictionary versioning

**Step 3: Audio Pipeline**
- Decouple audio generation from synthesis
- Create plugin architecture for synthesizers (Klatt, MBROLA, etc.)
- Add buffering and streaming support

**Step 4: API Layer**
- Implement context-based API
- Add async/await support
- Create event system for real-time control

---

### Phase 4: Build System Consolidation (Weeks 13-14)
**Goal**: Single, modern build system

#### Tasks
1. **Complete CMake Migration**
   - Remove `configure.ac`, `Makefile.am`
   - Add proper target exports
   - Create package config files

2. **Dependency Management**
   ```json
   // vcpkg.json
   {
     "name": "espeak-ng",
     "version": "2.0.0",
     "dependencies": [
       "pcaudiolib",
       {
         "name": "sonic",
         "optional": true
       }
     ]
   }
   ```

3. **CI/CD Enhancements**
   - Add sanitizers (ASan, UBSan, MSan)
   - Cross-compilation testing (ARM, RISC-V)
   - Performance benchmarks
   - Fuzzing integration

---

### Phase 5: AI-Assisted Testing Framework (Weeks 15-18)
**Goal**: Modern testing with AI validation

#### Testing Infrastructure

**1. Phoneme Accuracy Validator**
```python
# tests/ai_phoneme_validator.py
class PhonemeValidator:
    def validate_pronunciation(self, language: str, text: str, 
                              expected_ipa: str) -> ValidationResult:
        """
        Use AI to compare espeak phoneme output against expected IPA
        """
        actual_phonemes = espeak.get_phonemes(text, language)
        similarity = self.ai_compare(actual_phonemes, expected_ipa)
        return ValidationResult(
            passed=similarity > 0.95,
            similarity=similarity,
            suggestions=self.ai_suggest_fixes(actual_phonemes, expected_ipa)
        )
```

**2. Audio Regression Testing**
- Hash-based audio fingerprinting
- Voice consistency checks
- Performance regression detection

**3. Comprehensive Fuzzing**
- SSML fuzzing (expand existing)
- Phoneme input fuzzing
- Dictionary fuzzing
- Edge case text fuzzing

**4. Language Coverage**
- Automated tests for top 20 languages by usage
- Phoneme inventory validation
- Dictionary coverage analysis

---

### Phase 6: Language Quality Foundation (Weeks 19-20)
**Goal**: Prepare infrastructure for Phase 2 (Language Quality)

#### Language Metadata System
Extend `espeak-ng-data/lang/<family>/<code>` format:
```
name French
language fr
maintainer contact@example.com
quality verified|community|experimental
native_speakers_reviewed true
last_reviewed 2024-01-15
test_coverage 85%
```

#### Quality Metrics
- Dictionary coverage percentage
- Phoneme inventory completeness
- Pronunciation accuracy scores
- Performance benchmarks per language

#### Documentation
- `docs/language-development.md` - Guide for contributors
- `docs/phoneme-reference.md` - IPA mapping reference
- `docs/testing-guide.md` - How to write language tests

---

## New API Design

### Context-Based API (Phase 3)

**Initialization**:
```c
// Current API (global state)
espeak_Initialize(AUDIO_OUTPUT_PLAYBACK, 0, NULL, 0);
espeak_SetVoiceByName("en");
espeak_SetParameter(espeakRATE, 150, 0);

// New API (context-based)
espeak_context_t *ctx;
espeak_status status = espeak_create_context(&ctx);
espeak_config_t config = {
    .voice = "en",
    .rate = 150,
    .pitch = 50,
    .enable_ssml = true
};
espeak_configure(ctx, &config);
```

**Async/Streaming**:
```c
// Async synthesis
typedef void (*espeak_callback_t)(const audio_buffer_t *buffer, 
                                   void *user_data);

espeak_status espeak_speak_async(espeak_context_t *ctx,
                                 const char *text,
                                 espeak_callback_t callback,
                                 void *user_data);

// Real-time control
espeak_status espeak_stop(espeak_context_t *ctx);
espeak_status espeak_pause(espeak_context_t *ctx);
espeak_status espeak_set_parameter(espeak_context_t *ctx,
                                    espeak_param_t param,
                                    int value);
```

**Event System**:
```c
typedef enum {
    ESPEAK_EVENT_PHONEME,
    ESPEAK_EVENT_WORD,
    ESPEAK_EVENT_SENTENCE,
    ESPEAK_EVENT_MARK
} espeak_event_type_t;

typedef struct {
    espeak_event_type_t type;
    int position;
    const char *data;
} espeak_event_t;

espeak_status espeak_set_event_callback(espeak_context_t *ctx,
                                        void (*callback)(espeak_event_t));
```

---

## Risk Mitigation

### Breaking Changes
- **API Version**: Bump to 2.0.0
- **Compatibility Layer**: Optional shim for old API (deprecated)
- **Migration Guide**: Document all breaking changes

### Performance
- **Benchmarks**: Establish baseline before changes
- **Profiling**: Regular performance testing
- **Optimization**: Keep critical path lean

### Testing
- **Comprehensive Coverage**: Every change must have tests
- **Regression Suite**: All 100+ languages tested
- **Manual Testing**: Validate with screen reader users

---

## Success Metrics

### Technical Debt
- [ ] All FIXME comments resolved or documented
- [ ] Zero compiler warnings with `-Wall -Wextra`
- [ ] No unsafe string operations
- [ ] <5% global state (vs ~30 currently)

### Architecture
- [ ] All files <500 lines
- [ ] Clear module boundaries
- [ ] Documented APIs with examples
- [ ] Unit test coverage >80%

### Performance
- [ ] <10ms latency for first phoneme
- [ ] Support 800+ WPM speech rates
- [ ] Memory usage <50MB
- [ ] Real-time synthesis on low-end hardware

### Quality
- [ ] All 100+ languages have automated tests
- [ ] Phoneme accuracy >95% for major languages
- [ ] Zero crashes in fuzzing
- [ ] CI passes on all platforms

---

## Next Steps

1. **Immediate**: Review and approve roadmap
2. **Week 1**: Begin Phase 1 documentation
3. **Resource Planning**: Identify contributors for each phase
4. **Funding**: Consider applying for grants (accessibility focus)

---

## Resources

- [Original Article - Motivation](link-to-article)
- [eSpeak NG Wiki](https://github.com/espeak-ng/espeak-ng/wiki)
- [Current Issues](https://github.com/espeak-ng/espeak-ng/issues)
- [Mailing List](https://groups.io/g/espeak-ng)

---

**Notes**: This is a living document. Update as work progresses and priorities shift.
