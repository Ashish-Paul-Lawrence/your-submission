# NOTEBOOK.md

## Entry 1 – Obtaining the FLORES Corpus

**Hypothesis**

The FLORES dataset can be downloaded directly using the Hugging Face `datasets` library and exported to text files.

**Experiment**

* Installed the `datasets` package.
* Tried:

  ```
  load_dataset("facebook/flores", lang)
  ```

**Result**

Failed immediately because the dataset is gated.

Created a Hugging Face account, generated a read token, and authenticated with:

```
hf auth login
```

The login succeeded, but the dataset was still inaccessible because access to the repository itself had not yet been granted.

**Dead End**

Tried using `Muennighoff/flores200` instead.

This failed with:

```
RuntimeError: Dataset scripts are no longer supported
```

because the latest `datasets` library no longer supports dataset loading scripts.

**Revision**

After obtaining access to the official FLORES dataset, extracted only the four required languages (English, Hindi, Malayalam and Tamil) and saved both the `dev` and `devtest` splits as UTF-8 text files.

---

## Entry 2 – Running the Fertility Script

**Hypothesis**

The provided `fertility.py` script should give a reasonable comparison between English and Indic languages.

**Experiment**

Ran the script using GPT-2.

**Observation**

English produced approximately:

```
1.23 tokens/word
```

Hindi produced:

```
7.52 tokens/word
```

The difference looked suspiciously large.

**Initial assumption (incorrect)**

My first thought was that Hindi simply required more tokens because it uses a different script.

**Revision**

After examining GPT-2 more closely, I realised GPT-2 is primarily trained on English text and performs byte-level BPE tokenization.

Therefore the tokenizer—not necessarily the language—was responsible for most of the difference.

---

## Entry 3 – Trying a Multilingual Tokenizer

**Hypothesis**

If the large fertility gap is caused by GPT-2's tokenizer, a multilingual tokenizer should reduce it.

**Experiment**

Used

```
hf:xlm-roberta-base
```

instead of GPT-2.

**Result**

Hindi fertility dropped dramatically from roughly

```
7.5
```

to around

```
1.5 tokens/word
```

Malayalam and Tamil also became much closer to English.

**Conclusion**

The experiment confirmed that tokenizer choice has a much larger effect than the language itself.

This became one of the strongest pieces of evidence in the report.

---

## Entry 4 – Auditing the Code

**Hypothesis**

The supplied implementation might contain small bugs.

**Experiment**

Read through every function rather than assuming the implementation was correct.

**Findings**

Found several issues:

* fertility averaged sentence ratios instead of computing a corpus ratio
* `split(" ")` fails with repeated spaces or tabs
* text was converted to lowercase before tokenization
* tokens per character is misleading across Unicode scripts

**Unexpected finding**

Unicode NFC normalization initially looked suspicious because it modifies the input text.

After reading about Unicode normalization I concluded it actually improves consistency and should not be removed.

This was an example of something that looked like a bug but was not.

---

## Entry 5 – Choosing a Better Metric

**Initial hypothesis**

Tokens per word should be sufficient for comparing languages.

**Experiment**

Compared:

* tokens/word
* tokens/character
* tokens/parallel sentence

**Observation**

Characters vary significantly across scripts.

Whitespace word counts are inconsistent across languages.

Parallel sentences keep semantic content approximately constant because every FLORES sentence is a translation of the same source sentence.

**Revision**

Decided that average tokens per parallel sentence is the most meaningful metric for routing and inference-cost estimation.

---

## Entry 6 – KV Cache Calculations

**Hypothesis**

The benchmark behaviour should be explainable using the model specification.

**Experiment**

Calculated KV cache memory per token using:

* 28 layers
* 8 KV heads
* head dimension 128
* FP16 precision

Predicted:

* 114,688 bytes/token
* approximately 448 MiB per 4096-token sequence
* roughly 45 concurrent sequences on a 24 GB L4 GPU

**Observation**

The benchmark began showing scheduler preemption much earlier (around 32 requests).

**Revision**

Realised the theoretical calculation ignores allocator fragmentation, CUDA graph memory, runtime buffers and scheduler overhead.

The arithmetic correctly predicts the trend, but not the exact limit.

---

## Entry 7 – Investigating Throughput

**Hypothesis**

Higher batch size should continue increasing throughput.

**Experiment**

Examined the long-context benchmark rows.

**Observation**

Throughput increased until batch 24 but then decreased:

```
24 -> 1607 tok/s
32 -> 1384 tok/s
48 -> 1298 tok/s
```

At the same time:

* KV utilization reached approximately 97%
* preempted sequences increased from 0 to 7 and then 23

**Revision**

The bottleneck was no longer GPU compute.

The system became KV-cache limited.

The recommended deployment change became limiting long-context concurrency instead of increasing batch size.

---

## Entry 8 – Discovering the REPORT_v0 Mistake

**Hypothesis**

REPORT_v0's conclusion should be reproducible from the benchmark.

**Experiment**

Repeated the throughput calculations.

**Result**

Could not reproduce the reported value of approximately 3200 tok/s.

After checking the benchmark columns carefully, I realised the report had interpreted `reported_tok_s` as throughput per request even though it already represents aggregate throughput across the batch.

That single misunderstanding explained both incorrect conclusions:

* longer prompts supposedly improve throughput
* batch 48 supposedly reaches 3200 tok/s

---

## Entry 9 – Product Recommendation

**Hypothesis**

Fine-tuning would produce the best conversational style.

**Experiment**

Estimated the available reviewer time.

Reviewer availability:

* 10 hours/week
* 2 weeks
* approximately 600 reviewed examples

**Observation**

The available review budget was far too small for creating a reliable multilingual SFT dataset.

**Revision**

Changed the recommendation from SFT to a lightweight inference-time rewriter.

The rewriter could be evaluated quickly, deployed independently, and abandoned if reviewer preference remained below the target threshold.

---

## Final Reflection

Several of my initial assumptions turned out to be incorrect.

The largest surprises were:

* GPT-2's poor performance on Indic scripts compared with a multilingual tokenizer.
* The supplied fertility implementation contained subtle statistical errors.
* The benchmark throughput collapse was caused by KV-cache saturation rather than insufficient GPU compute.
* A single misinterpretation of one benchmark column completely changed the conclusions in REPORT_v0.

Overall, most of the investigation consisted of validating assumptions with measurements rather than accepting the initial results at face value.
