# AI-Assisted Testing Framework

**Status**: Design Phase  
**Last Updated**: 2026-02-02

---

## Overview

Modern AI tools can significantly improve testing quality for espeak-ng, particularly for:
- Phoneme accuracy validation
- Pronunciation regression detection
- Dictionary coverage analysis
- Language quality assessment

This document outlines the framework design and implementation plan.

---

## Framework Architecture

```
tests/
├── ai_framework/
│   ├── __init__.py
│   ├── validators/           # AI validation modules
│   │   ├── phoneme_validator.py
│   │   ├── pronunciation_validator.py
│   │   └── quality_assessor.py
│   ├── generators/           # Test data generators
│   │   ├── edge_cases.py
│   │   ├── language_corpus.py
│   │   └── fuzzing_data.py
│   ├── benchmarks/           # Performance benchmarks
│   │   ├── latency_tests.py
│   │   └── throughput_tests.py
│   └── utils/
│       ├── espeak_wrapper.py
│       ├── phoneme_utils.py
│       └── report_generator.py
├── test_data/
│   ├── reference_audio/      # Reference pronunciations
│   ├── expected_phonemes/    # Expected IPA output
│   └── corpora/              # Test corpora per language
└── results/                  # Test results storage
```

---

## Core Components

### 1. Phoneme Validator

**Purpose**: Validate espeak phoneme output against expected IPA

```python
# tests/ai_framework/validators/phoneme_validator.py

from dataclasses import dataclass
from typing import List, Optional, Dict
import openai  # or local LLM

@dataclass
class PhonemeValidationResult:
    text: str
    language: str
    espeak_phonemes: str
    expected_ipa: str
    similarity_score: float
    is_valid: bool
    suggestions: List[str]
    phoneme_breakdown: Dict[str, str]

class AIPhonemeValidator:
    """
    Uses AI to validate espeak phoneme output quality.
    """
    
    def __init__(self, model: str = "gpt-4"):
        self.model = model
        self.threshold = 0.95
    
    def validate(self, 
                 text: str, 
                 language: str,
                 expected_ipa: Optional[str] = None) -> PhonemeValidationResult:
        """
        Validate espeak phoneme output for given text.
        
        If expected_ipa is not provided, AI will generate expected pronunciation.
        """
        # Get espeak output
        espeak_phonemes = self._get_espeak_phonemes(text, language)
        
        # Get expected pronunciation
        if expected_ipa is None:
            expected_ipa = self._ai_generate_expected(text, language)
        
        # Compare using AI
        similarity = self._ai_compare_phonemes(
            espeak_phonemes, 
            expected_ipa, 
            language
        )
        
        # Generate suggestions if mismatch
        suggestions = []
        if similarity < self.threshold:
            suggestions = self._ai_suggest_fixes(
                text, 
                espeak_phonemes, 
                expected_ipa
            )
        
        return PhonemeValidationResult(
            text=text,
            language=language,
            espeak_phonemes=espeak_phonemes,
            expected_ipa=expected_ipa,
            similarity_score=similarity,
            is_valid=similarity >= self.threshold,
            suggestions=suggestions,
            phoneme_breakdown=self._breakdown_phonemes(espeak_phonemes)
        )
    
    def _ai_compare_phonemes(self, 
                             espeak_out: str, 
                             expected: str,
                             language: str) -> float:
        """
        Use AI to compare phoneme sequences.
        
        Returns similarity score 0.0 to 1.0
        """
        prompt = f"""
        Compare these two phoneme transcriptions for {language}:
        
        Espeak output: {espeak_out}
        Expected IPA: {expected}
        
        Consider:
        - Phonetic equivalence (e.g., 'a' vs 'ɑ' might be equivalent)
        - Stress placement
        - Allophone variation
        - Minor dialect differences
        
        Return a similarity score from 0.0 to 1.0, where:
        - 1.0 = Perfect match
        - 0.9+ = Minor acceptable differences
        - 0.8+ = Noticeable but understandable
        - <0.8 = Significant issues
        
        Also provide brief reasoning.
        """
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}]
        )
        
        # Parse response to extract score
        return self._parse_similarity_response(response)
    
    def _ai_suggest_fixes(self, 
                          text: str,
                          espeak_out: str, 
                          expected: str) -> List[str]:
        """
        Get AI suggestions for fixing pronunciation.
        """
        prompt = f"""
        The espeak-ng TTS system pronounced "{text}" as: {espeak_out}
        
        But it should be: {expected}
        
        Suggest specific fixes for the espeak-ng dictionary or rules files.
        Consider:
        - Which phonemes are wrong?
        - Is it a dictionary entry issue or a rule issue?
        - What would the fix look like?
        
        Provide concrete suggestions that a developer could implement.
        """
        
        response = openai.ChatCompletion.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}]
        )
        
        return self._parse_suggestions(response)
```

---

### 2. Pronunciation Regression Detector

**Purpose**: Detect changes in pronunciation between versions

```python
# tests/ai_framework/validators/regression_detector.py

class RegressionDetector:
    """
    Detects pronunciation changes between espeak-ng versions.
    """
    
    def __init__(self, baseline_version: str):
        self.baseline = self._load_baseline(baseline_version)
    
    def detect_regressions(self, 
                          test_corpus: List[str],
                          language: str) -> List[RegressionReport]:
        """
        Compare current output against baseline.
        """
        regressions = []
        
        for text in test_corpus:
            current = self._get_phonemes(text, language)
            baseline = self.baseline.get(text)
            
            if baseline and current != baseline:
                # Use AI to determine if change is regression or improvement
                assessment = self._ai_assess_change(
                    text, baseline, current, language
                )
                
                if assessment.is_regression:
                    regressions.append(RegressionReport(
                        text=text,
                        old_phonemes=baseline,
                        new_phonemes=current,
                        severity=assessment.severity,
                        explanation=assessment.explanation
                    ))
        
        return regressions
    
    def _ai_assess_change(self, 
                          text: str,
                          old: str,
                          new: str,
                          language: str) -> ChangeAssessment:
        """
        Use AI to determine if phoneme change is good or bad.
        """
        prompt = f"""
        In {language}, the pronunciation of "{text}" changed:
        - Old: {old}
        - New: {new}
        
        Is this change:
        a) A regression (worse pronunciation)
        b) An improvement (better pronunciation)
        c) Neutral (different but equally valid)
        
        Provide reasoning and classify severity if regression:
        - Critical: Unintelligible or completely wrong
        - High: Noticeably wrong
        - Medium: Slightly off
        - Low: Acceptable variation
        """
        
        # Call AI model and parse response
        ...
```

---

### 3. Language Quality Assessor

**Purpose**: Assess overall language support quality

```python
# tests/ai_framework/validators/quality_assessor.py

class LanguageQualityReport:
    """
    Comprehensive quality assessment for a language.
    """
    phoneme_coverage: float  # % of language phonemes supported
    dictionary_coverage: float  # % of common words in dictionary
    pronunciation_accuracy: float  # % of test cases correct
    stress_accuracy: float  # % correct stress placement
    intonation_quality: str  # Assessment of prosody
    overall_score: float
    recommendations: List[str]

class LanguageQualityAssessor:
    """
    Assess quality of language support in espeak-ng.
    """
    
    def assess_language(self, language_code: str) -> LanguageQualityReport:
        """
        Perform comprehensive quality assessment.
        """
        # 1. Check phoneme inventory
        phoneme_coverage = self._assess_phoneme_coverage(language_code)
        
        # 2. Test common words
        dictionary_coverage = self._test_dictionary_coverage(language_code)
        
        # 3. Test pronunciation accuracy
        pronunciation_accuracy = self._test_pronunciation_accuracy(language_code)
        
        # 4. Check stress placement
        stress_accuracy = self._test_stress_accuracy(language_code)
        
        # 5. Use AI for qualitative assessment
        qualitative = self._ai_qualitative_assessment(language_code)
        
        return LanguageQualityReport(
            phoneme_coverage=phoneme_coverage,
            dictionary_coverage=dictionary_coverage,
            pronunciation_accuracy=pronunciation_accuracy,
            stress_accuracy=stress_accuracy,
            intonation_quality=qualitative.intonation,
            overall_score=self._calculate_overall_score(
                phoneme_coverage,
                dictionary_coverage,
                pronunciation_accuracy
            ),
            recommendations=qualitative.recommendations
        )
    
    def _ai_qualitative_assessment(self, language_code: str):
        """
        Get AI assessment of language quality.
        """
        # Generate sample audio/text
        samples = self._generate_samples(language_code)
        
        prompt = f"""
        Assess the quality of text-to-speech for {language_code}.
        
        Sample output:
        {samples}
        
        Evaluate:
        1. Phoneme accuracy (are sounds correct?)
        2. Prosody/intonation (does it sound natural?)
        3. Rhythm and timing
        4. Common error patterns
        
        Provide:
- Overall quality rating (1-10)
- Specific issues found
- Recommendations for improvement
        """
        
        # Call AI and parse
        ...
```

---

## Test Data Generation

### 1. Edge Case Generator

```python
# tests/ai_framework/generators/edge_cases.py

class EdgeCaseGenerator:
    """
    Generates edge cases for testing.
    """
    
    def generate_for_language(self, language: str) -> List[TestCase]:
        """
        Generate comprehensive edge cases.
        """
        cases = []
        
        # Numbers
        cases.extend(self._number_cases(language))
        
        # Punctuation
        cases.extend(self._punctuation_cases())
        
        # Homographs
        cases.extend(self._homograph_cases(language))
        
        # Foreign words
        cases.extend(self._foreign_word_cases(language))
        
        # Special characters
        cases.extend(self._special_character_cases())
        
        # Use AI to generate additional edge cases
        cases.extend(self._ai_generate_edge_cases(language))
        
        return cases
    
    def _number_cases(self, language: str) -> List[TestCase]:
        """
        Generate number edge cases.
        """
        numbers = [
            "0", "1", "10", "100", "1000",
            "1234", "1000000", "3.14159",
            "1/2", "2024", "-50",
            "1st", "2nd", "3rd", "4th",
            "January 1, 2024", "12:30 PM"
        ]
        
        return [TestCase(text=n, category="number") for n in numbers]
    
    def _ai_generate_edge_cases(self, language: str) -> List[TestCase]:
        """
        Use AI to identify language-specific edge cases.
        """
        prompt = f"""
        For {language}, what are common text-to-speech challenges?
        
        Consider:
        - Irregular spellings
        - Loan words from other languages
        - Abbreviations and acronyms
        - Ambiguous pronunciations
        - Cultural proper nouns
        
        Provide 20 specific examples that would test a TTS system.
        """
        
        # Call AI and generate cases
        ...
```

---

### 2. Language Corpus Generator

```python
# tests/ai_framework/generators/language_corpus.py

class LanguageCorpusGenerator:
    """
    Generates representative text corpora for languages.
    """
    
    def generate_corpus(self, 
                       language: str,
                       size: int = 1000) -> List[str]:
        """
        Generate balanced test corpus.
        """
        corpus = []
        
        # Common words (frequency based)
        corpus.extend(self._common_words(language, count=size//4))
        
        # Sentences of varying complexity
        corpus.extend(self._sentences(language, count=size//4))
        
        # Named entities
        corpus.extend(self._named_entities(language, count=size//8))
        
        # Technical terms
        corpus.extend(self._technical_terms(language, count=size//8))
        
        # Fill rest with AI-generated sentences
        remaining = size - len(corpus)
        corpus.extend(self._ai_generate_sentences(language, remaining))
        
        return corpus
```

---

## Benchmarking

### Performance Tests

```python
# tests/ai_framework/benchmarks/latency_tests.py

import time
import statistics

class LatencyBenchmark:
    """
    Measure synthesis latency.
    """
    
    def benchmark_first_phoneme_latency(self, 
                                       test_texts: List[str],
                                       language: str) -> BenchmarkResult:
        """
        Measure time from text input to first audio output.
        """
        latencies = []
        
        for text in test_texts:
            start = time.perf_counter_ns()
            
            # Start synthesis with callback
            callback_triggered = threading.Event()
            
            def on_phoneme(phoneme):
                callback_triggered.set()
            
            espeak.speak_async(text, on_phoneme)
            callback_triggered.wait(timeout=5.0)
            
            end = time.perf_counter_ns()
            latencies.append((end - start) / 1_000_000)  # Convert to ms
        
        return BenchmarkResult(
            mean_latency_ms=statistics.mean(latencies),
            median_latency_ms=statistics.median(latencies),
            p95_latency_ms=statistics.quantiles(latencies, n=20)[18],
            p99_latency_ms=statistics.quantiles(latencies, n=100)[98],
            max_latency_ms=max(latencies)
        )
    
    def benchmark_throughput(self,
                           text: str,
                           duration_seconds: int) -> ThroughputResult:
        """
        Measure words per minute synthesis rate.
        """
        word_count = len(text.split())
        start = time.time()
        
        words_processed = 0
        while time.time() - start < duration_seconds:
            espeak.speak(text)
            words_processed += word_count
        
        elapsed = time.time() - start
        wpm = (words_processed / elapsed) * 60
        
        return ThroughputResult(
            words_per_minute=wpm,
            real_time_factor=words_processed / elapsed
        )
```

---

## Integration with CI/CD

### GitHub Actions Integration

```yaml
# .github/workflows/ai-testing.yml

name: AI-Assisted Testing

on: [pull_request]

jobs:
  ai-phoneme-validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build espeak-ng
        run: |
          mkdir build && cd build
          cmake ..
          make -j$(nproc)
      
      - name: Run AI phoneme validation
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          cd tests/ai_framework
          python -m pytest validators/test_phoneme_validator.py \
            --languages=en,fr,de,es \
            --threshold=0.95
      
      - name: Check for regressions
        run: |
          python -m ai_framework.regression_detector \
            --baseline=main \
            --languages=all
      
      - name: Generate quality report
        run: |
          python -m ai_framework.quality_assessor \
            --output=quality-report.md
      
      - name: Comment PR with results
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('quality-report.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: report
            });
```

---

## Local LLM Alternative

For projects without API budget, use local models:

```python
# tests/ai_framework/llm_local.py

from transformers import AutoModelForCausalLM, AutoTokenizer

class LocalLLMValidator:
    """
    Use local LLM for validation (no API costs).
    """
    
    def __init__(self, model_name: str = "microsoft/DialoGPT-medium"):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForCausalLM.from_pretrained(model_name)
    
    def validate_phonemes(self, 
                          espeak_out: str,
                          expected: str) -> float:
        """
        Local validation without API calls.
        """
        # Simpler rule-based approach for local use
        # Or use smaller local model
        ...
```

---

## Implementation Plan

### Phase 1: Basic Framework (Weeks 1-2)
- [ ] Create directory structure
- [ ] Implement espeak wrapper module
- [ ] Create basic phoneme comparison
- [ ] Set up pytest infrastructure

### Phase 2: AI Integration (Weeks 3-4)
- [ ] Implement AI phoneme validator
- [ ] Create regression detector
- [ ] Add quality assessment
- [ ] Set up CI integration

### Phase 3: Test Data (Weeks 5-6)
- [ ] Generate edge cases for top 20 languages
- [ ] Create reference pronunciation database
- [ ] Implement corpus generators

### Phase 4: Benchmarks (Weeks 7-8)
- [ ] Add latency benchmarks
- [ ] Add throughput benchmarks
- [ ] Create performance regression detection
- [ ] Set up performance dashboards

### Phase 5: Integration (Weeks 9-10)
- [ ] Full CI/CD integration
- [ ] PR quality checks
- [ ] Automated reporting
- [ ] Documentation

---

## Cost Considerations

### API Usage Estimates

**Per PR Validation**:
- ~1000 API calls for basic validation
- ~$0.50 per PR (GPT-4)

**Full Language Assessment**:
- ~5000 API calls per language
- ~$2.50 per language (GPT-4)

**Total Monthly** (50 PRs, 100 languages):
- ~$275/month

### Cost Reduction Strategies
1. **Caching**: Cache AI responses for identical queries
2. **Tiered Testing**: Run full AI tests only on significant changes
3. **Local Models**: Use smaller models for initial screening
4. **Sampling**: Test subset of languages per PR

---

## Benefits

1. **Objective Quality Metrics**: AI provides consistent assessment
2. **Catch Regressions Early**: Automated detection before merge
3. **Language Coverage**: Can test all 100+ languages automatically
4. **Developer Guidance**: Specific suggestions for fixes
5. **Documentation**: Generated reports show quality over time

---

## Risks and Mitigations

### Risk: AI Hallucination
**Mitigation**: 
- Use rule-based validation for critical paths
- Cross-reference with multiple AI queries
- Human review of flagged issues

### Risk: API Costs
**Mitigation**:
- Implement caching aggressively
- Use local models for routine checks
- Budget alerts and limits

### Risk: False Positives
**Mitigation**:
- Tune thresholds carefully
- Allowlist acceptable variations
- Human override capability

---

**Next Steps**: Begin Phase 1 implementation after technical debt Phase 2.
