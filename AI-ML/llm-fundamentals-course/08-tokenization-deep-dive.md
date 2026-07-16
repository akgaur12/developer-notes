# Chapter 8: Tokenization Deep Dive

*Chapter 7 drew a box labeled "Tokenizer" at the very start of the generation pipeline and moved on. This chapter opens that box: the algorithm that decides what a "token" even is, and why that decision quietly controls your context window, your latency, and your API bill.*

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain why LLMs tokenize text into subwords rather than characters or whole words
- Perform Byte Pair Encoding (BPE) merges by hand on a toy corpus and explain the stopping condition
- Contrast BPE, WordPiece, SentencePiece, and Unigram tokenization at the level of "what does each optimize when choosing merges/splits"
- Explain why tiktoken and other tokenizers differ across model families, and why that matters when porting prompts between models
- Reason about the vocabulary-size trade-off between sequence length and embedding-matrix size
- Explain why token count, not character count, determines API cost and context usage — and why this varies by language
- Implement a minimal BPE tokenizer from scratch in Python

---

## Prerequisites for This Chapter

This chapter builds directly on **Chapter 7: LLM Architecture: Decoder-Only Models, KV Cache & RoPE**, where the full generation pipeline was introduced as:

```
Prompt → Tokenizer → Embeddings → Transformer Layers → Logits → Sampling → Generated Token
```

Chapter 7 treated the **Tokenizer** step as a black box that "turns text into tokens" so it could focus on everything downstream. It also established that the token embedding matrix has shape `[vocab_size, d_model]` — a fact that becomes important in this chapter's discussion of vocabulary size trade-offs. This chapter is entirely about that first box: what a token actually is, and the algorithm that decides where to cut a string into pieces.

---

## 1. Why Tokenization Exists At All

You might reasonably ask: why not just feed the model raw characters, or raw bytes? Or, at the other extreme, why not just split on whitespace and treat every whole word as one unit? Both extremes fail for concrete, mechanical reasons.

**Character-level tokenization fails on cost.** Recall from Chapter 5 that self-attention costs `O(n²)` in sequence length `n`. A sentence like "tokenization is expensive" is 26 characters but only 4–5 subword tokens under a real tokenizer. Character-level tokenization would multiply the effective sequence length by roughly 5–6x, and since attention cost is *quadratic*, that's a 25–36x increase in compute for the same text. Every paragraph you send an LLM would quietly cost dozens of times more compute.

**Whole-word tokenization fails on vocabulary and coverage.** English alone has hundreds of thousands of word forms once you count inflections ("run", "runs", "running", "ran"), and that's before touching typos, brand names, code identifiers, or any other language. A whole-word vocabulary large enough to cover realistic text would need to be enormous (blowing up the embedding matrix — more on this in Section 6), and it would still hit an **out-of-vocabulary (OOV)** wall the moment it saw a word it hadn't memorized — a misspelling, a new product name, a rare technical term. There would be no way to represent it at all, only a generic `<unk>` placeholder that throws away all the information in that word.

**Subword tokenization is the practical middle ground.** The insight: break text into pieces *smaller than words but larger than characters* — common whole words stay as single tokens ("the", "is"), while rare or unseen words decompose into recognizable pieces ("tokenization" → "token" + "ization"). This gives you:

- A **bounded, fixed vocabulary** (typically 30,000–200,000 entries) — no OOV wall, because in the worst case a word can always fall back to individual bytes/characters, which are always in vocabulary
- **Reasonable sequence lengths** — common text compresses to roughly 3–5 tokens per English word on average, not one token per character
- **Some semantic structure preserved** — related words ("token", "tokens", "tokenizer") often share subword pieces, giving the model a head start on generalization

Every algorithm in this chapter is a different recipe for deciding *which* subword pieces make it into that fixed vocabulary.

```
Character-level:  t-o-k-e-n-i-z-a-t-i-o-n   (12 tokens, cheap vocab, expensive sequences)
Word-level:       tokenization                (1 token, if it's in vocab — else <unk>, total information loss)
Subword-level:    token-ization                (2 tokens — bounded vocab, no OOV, decent length)
```

---

## 2. Byte Pair Encoding (BPE)

BPE is the tokenization algorithm behind GPT-2, GPT-3/4-family models (via tiktoken), and Llama. Despite powering some of the most complex software in the world, the algorithm itself is almost embarrassingly simple.

### 2.1 The algorithm

1. Start with a vocabulary of individual characters (or bytes) — every symbol that appears in your training corpus is a token.
2. Represent every word in the training corpus as a sequence of these base symbols, and count how often every *adjacent pair* of symbols occurs across the whole corpus.
3. Find the single most frequent pair, and **merge** it into one new symbol. Add that new symbol to the vocabulary.
4. Repeat steps 2–3 — recount pairs in the now-partially-merged corpus, merge the new most frequent pair — until you reach a target vocabulary size (a hyperparameter chosen in advance, e.g. 50,000 for GPT-2).

The core idea: greedily build up the *most statistically useful* multi-character chunks first, because they're the ones that appear most often across your data.

### 2.2 A fully worked example

Take a tiny toy corpus with word frequencies (this is the same illustrative example used in the original BPE-for-NLP paper by Sennrich et al.):

```
low       (×5)
lower     (×2)
newest    (×6)
widest    (×3)
```

Represent each word as a sequence of characters, with an end-of-word marker `</w>` so the model can tell "est" at the end of a word apart from "est" in the middle of one:

```
l o w </w>            ×5
l o w e r </w>         ×2
n e w e s t </w>       ×6
w i d e s t </w>       ×3
```

**Merge 1** — count every adjacent pair across all words (weighted by word frequency). The pairs `(e,s)`, `(s,t)`, and `(t,</w>)` are all tied at count 9 (6 from "newest" + 3 from "widest" each). Take the first tie by convention: merge `(e, s)` → `es`.

```
l o w </w>              ×5
l o w e r </w>           ×2
n e w es t </w>          ×6
w i d es t </w>          ×3
```

**Merge 2** — recount. `(es, t)` is now the most frequent pair at count 9. Merge `es t` → `est`.

```
l o w </w>              ×5
l o w e r </w>           ×2
n e w est </w>           ×6
w i d est </w>           ×3
```

**Merge 3** — `(est, </w>)` is now most frequent at count 9. Merge → `est</w>`.

```
l o w </w>              ×5
l o w e r </w>           ×2
n e w est</w>            ×6
w i d est</w>            ×3
```

**Merge 4** — count again. `(l, o)` and `(o, w)` are now tied at count 7 (5 from "low" + 2 from "lower" each). Merge `l o` → `lo`.

```
lo w </w>               ×5
lo w e r </w>            ×2
n e w est</w>            ×6
w i d est</w>            ×3
```

**Merge 5** — `(lo, w)` is now the top pair at count 7. Merge → `low`.

```
low </w>                ×5
low e r </w>             ×2
n e w est</w>            ×6
w i d est</w>            ×3
```

After 5 merges, the vocabulary has grown from 11 base characters (`l,o,w,e,r,n,s,t,i,d,</w>`) to 16 symbols, and the corpus already looks noticeably more compressed: `low` and `est</w>` are now single tokens. Keep repeating this process — merging `(n,e)`→`ne`, then `(ne,w)`→`new`, and so on — and eventually `newest` collapses to two tokens (`new`+`est</w>`) instead of six characters. Real BPE tokenizers run this loop tens of thousands of times over a training corpus of billions of characters, not four toy words, but the mechanism is *exactly* this: count adjacent pairs, merge the most frequent, repeat.

### 2.3 What "applying" the tokenizer means at inference time

Training BPE produces an ordered list of merge rules (a "merge table"). Applying it to new text is deterministic: split the text into base characters/bytes, then apply the learned merges *in the order they were learned*, same as in training. This is why a trained tokenizer never needs to see your specific input text before — it just replays its fixed merge table.

### 2.4 Illustrative Python sketch

```python
from collections import Counter

def get_pair_counts(corpus: dict[tuple[str, ...], int]) -> Counter:
    pairs = Counter()
    for word, freq in corpus.items():
        for a, b in zip(word, word[1:]):
            pairs[(a, b)] += freq
    return pairs

def merge_pair(pair, corpus):
    a, b = pair
    merged = {}
    for word, freq in corpus.items():
        new_word, i = [], 0
        while i < len(word):
            if i < len(word) - 1 and word[i] == a and word[i + 1] == b:
                new_word.append(a + b)
                i += 2
            else:
                new_word.append(word[i])
                i += 1
        merged[tuple(new_word)] = merged.get(tuple(new_word), 0) + freq
    return merged

def train_bpe(corpus: dict[tuple[str, ...], int], num_merges: int):
    merges = []
    for _ in range(num_merges):
        pairs = get_pair_counts(corpus)
        if not pairs:
            break
        best = max(pairs, key=pairs.get)
        corpus = merge_pair(best, corpus)
        merges.append(best)
    return merges, corpus
```

This is BPE in roughly 20 lines — everything else in a production tokenizer (Section 5's tiktoken, regex-based pre-splitting, byte-level fallbacks) is engineering around this core loop, not a different idea.

---

## 3. WordPiece

WordPiece is the tokenization algorithm behind BERT. It follows the same bottom-up *merging* shape as BPE, with one key difference in **how it picks which pair to merge**.

BPE greedily picks the pair with the highest raw **frequency**. WordPiece instead picks the pair that gives the highest **likelihood gain** for the training corpus — roughly, it scores a candidate merge `(a, b)` by:

```
score(a, b) = count(a, b) / (count(a) × count(b))
```

This normalizes by how frequent `a` and `b` are *individually*. A pair like `("t", "h")` might have a high raw count simply because both `t` and `h` are individually extremely common letters — but that doesn't mean `"th"` is a *meaningfully* better single unit than treating them separately. WordPiece's score favors pairs that co-occur *more often than you'd expect by chance* given their individual frequencies, which tends to surface more linguistically meaningful subwords rather than just the statistically loudest ones.

In practice, BPE and WordPiece produce broadly similar-looking vocabularies (both are subword, both merge bottom-up), and the distinction matters more for research nuance than for how you use these models day to day — the important takeaway is simply that **not all merge-based tokenizers optimize the same objective**, and that difference is one reason BERT's vocabulary looks slightly different in character from GPT-2's even on similar training data.

---

## 4. SentencePiece

BPE and WordPiece, as described above, assume you can pre-split text on whitespace before tokenizing words into subwords. That assumption breaks for languages like Japanese, Chinese, or Thai, which don't use whitespace to separate words at all.

**SentencePiece** solves this by treating tokenization as an operation over the *raw, unsegmented* input string — including whitespace itself as a regular character to be modeled, conventionally displayed as `▁` (U+2581, "lower one eighth block"). Instead of "split on spaces, then subword-tokenize each word," SentencePiece subword-tokenizes the *entire raw string, whitespace included*, from the start.

Concretely, the English sentence `"Hello world"` might tokenize to `["▁Hello", "▁world"]` — the `▁` marks "this token starts a new word (was preceded by a space in the original text)." This has two practical benefits:

1. **Language-agnostic**: it works identically whether or not the language uses whitespace as a word boundary, since whitespace is just another character in the model rather than a special pre-processing rule.
2. **Fully reversible (lossless)**: because whitespace is encoded *inside* the tokens rather than stripped out beforehand, you can always reconstruct the exact original string — including its exact spacing — by concatenating tokens and replacing `▁` with a space. Naive whitespace-splitting tokenizers often can't guarantee this.

SentencePiece is a training/serialization *framework* that can run either the BPE algorithm or the Unigram algorithm (Section 5 below) underneath it — it's the "how the raw text is fed in" layer, decoupled from "which merge/split algorithm runs on it." T5, Llama, and many multilingual models use SentencePiece specifically for this whitespace-as-character, fully-reversible property.

---

## 5. Unigram Language Model Tokenization

Every algorithm so far has worked **bottom-up**: start from characters, merge upward toward a target vocabulary size. Unigram tokenization works **top-down**, and optimizes a genuinely different objective.

1. Start with a large candidate vocabulary (e.g. every substring that appears often enough in the corpus, could easily be 1,000,000+ candidates).
2. Fit a **unigram language model** — a probability for each vocabulary entry, assuming tokens are drawn independently — that maximizes the likelihood of the training corpus under some tokenization of it.
3. For each vocabulary entry, compute how much the corpus's overall likelihood would *drop* if that entry were removed from the vocabulary.
4. Remove the entries whose removal hurts the likelihood the *least* (i.e., the least useful tokens), shrinking the vocabulary by some percentage.
5. Repeat steps 2–4 until the vocabulary reaches the target size.

The practical difference this creates: because Unigram tokenization is a genuine probabilistic model rather than a fixed merge table, encoding a string isn't necessarily forced down one single path — multiple candidate segmentations exist, and the model can pick (or sample among) the most probable one. This gives frameworks built on Unigram (SentencePiece supports it as an alternative to BPE) a form of built-in **subword regularization**: during training, you can deliberately sample a slightly different-but-valid segmentation of the same word on different training passes, which acts like a data augmentation that makes the downstream model more robust to how a given word happens to get sliced.

---

## 6. tiktoken and Tokenizer Fragmentation Across Model Families

**tiktoken** is OpenAI's tokenizer library — a fast, byte-level BPE implementation used by the GPT model family. It's worth knowing by name for one very practical reason: **every model family ships its own tokenizer and vocabulary, trained on its own data, and they are not interchangeable.**

This has a concrete, easy-to-miss consequence: the *same* piece of text produces a *different* number of tokens depending on which model's tokenizer processes it. A prompt that costs 800 tokens against a GPT-family tokenizer might cost 750 or 900 tokens against a Llama-family tokenizer — because the two vocabularies were trained on different corpora with different merge tables, so they draw the subword boundaries in different places. This is why:

- You cannot reuse one model's token count/cost estimate for a different model family — you have to re-tokenize with the target model's actual tokenizer to get an accurate count.
- Context-window limits (Chapter 7) are stated in that model's *own* tokens — "128K context" for one model and "128K context" for another can hold measurably different amounts of raw text if their tokenizers segment text at different average efficiency.
- Fine-tuning or building tooling around a specific model's tokenizer (e.g. a custom vocabulary extension) does not transfer to a different model family without redoing that work against the new tokenizer.

---

## 7. Vocabulary Design Trade-offs

Chapter 7 established that the token embedding matrix has shape `[vocab_size, d_model]`, and that the same matrix (transposed) is typically reused as the final projection to logits over the vocabulary. Vocabulary size is therefore not a free knob — it trades off two costs directly against each other:

**Bigger vocabulary →**
- Shorter token sequences per input (more text compresses into fewer tokens, since more common multi-character chunks get their own single token)
- Cheaper attention per input (recall the `O(n²)` cost in sequence length `n` — fewer tokens means quadratically less attention compute for the same text)
- **But**: a larger embedding matrix and a larger final output projection — more parameters, more memory, and a more expensive softmax over the vocabulary at every generation step

**Smaller vocabulary →**
- Smaller embedding/output matrices, less memory
- **But**: longer token sequences for the same text (words get sliced into more, smaller pieces), which directly costs more attention compute and eats more of the fixed context window per input

There is no universally "correct" vocabulary size — it's a genuine engineering trade-off, and real model families land at noticeably different points: GPT-2 used roughly 50K tokens, many modern multilingual models use 100K–250K+ tokens (a bigger vocabulary helps spread coverage across many languages without each one degrading into very long token sequences, at the cost of a larger embedding table).

---

## 8. Context Length, Token Cost, and Token Efficiency

Because LLM APIs bill **per token**, and context windows are measured **in tokens** (not characters or words), tokenization efficiency has a direct, measurable effect on your cost and your usable context budget — not just an abstract implementation detail.

**Rare and unusual text "wastes" tokens.** A common English word like "the" is almost always one token. A rare, unusual, or invented word gets sliced into several subword pieces, because the tokenizer's merge table never learned a single-token shortcut for something it saw rarely (or never) during training. The same logic applies to code identifiers, product names, and typos — anything statistically rare in the tokenizer's training data costs proportionally more tokens per character.

**Non-English text is often measurably less token-efficient than English**, for a structural reason, not a linguistic one: most widely-used general-purpose tokenizers are trained on corpora that are disproportionately English (or Latin-script) text. As a result, common English words earn dedicated single-token merges, while text in other scripts — Hindi, Japanese, Arabic — ends up represented with more, smaller subword or even byte-level fallback tokens for the *same amount of meaning*, simply because that script's common word-forms didn't appear often enough in training to earn efficient merges of their own. Reasoning through a concrete case: the English sentence "The quick brown fox jumps over the lazy dog" (9 short, extremely common English words) will typically tokenize to somewhere close to one token per word under a GPT-family tokenizer. An equivalent-length sentence in a script with much less representation in the tokenizer's training data can easily tokenize to two, three, or more tokens per word — meaning the *same idea, expressed in a different language, can cost meaningfully more tokens and more of the context window* to represent, purely as an artifact of tokenizer training data composition, not any property of the language itself.

**Practical implication**: if you're building multilingual products or working with a lot of code/identifiers, budget extra context and cost headroom rather than assuming token counts scale evenly with character counts across languages or domains.

---

## Real-World Scenario

A team ships a customer-support chatbot that works beautifully in QA (English-only test scripts) with a fixed context budget: system prompt + retrieved knowledge-base chunks + conversation history, capped at a hard 8,000-token limit before the model call fails outright with a context-length error.

Three weeks after launch, the team expands to Spanish- and Hindi-speaking markets — same system prompt, same knowledge base (professionally translated), same 8,000-token cap. Within days, Hindi-language conversations start silently truncating retrieved context or hitting hard context-length errors mid-conversation, even though the *translated text is roughly the same length in words* as the English original. The root cause, once traced: the tokenizer in use was trained overwhelmingly on English/Latin-script web text. The same knowledge-base passage that took ~180 tokens in English took over 340 tokens in Hindi — nearly double — purely because Hindi words fragment into many more subword/byte-level tokens under this particular tokenizer's merge table. The fixed 8,000-token budget, sized against English-language testing, was never actually a fixed *content* budget once other languages entered the mix.

The fix wasn't a smarter prompt — it was re-measuring the actual token cost of representative content **per target language** using the real tokenizer, and either increasing the context budget for non-English markets, trimming retrieved-context size per language, or choosing a tokenizer/model with better multilingual token efficiency for that expansion.

---

## Best Practices

- **Always measure token counts with the target model's actual tokenizer**, never estimate from character or word counts — the ratio varies by model family, language, and content type.
- **Budget extra context headroom for non-English and code-heavy content**, since tokenizers trained on English-skewed corpora are measurably less efficient on other scripts and on identifiers/rare terms.
- **Re-tokenize (don't reuse cached token counts) whenever you swap model families** — a token-count budget computed against one tokenizer does not transfer to another.
- **Test context-window and cost assumptions against real production-like content**, not just clean English QA scripts, especially before a multilingual or code-heavy launch.
- **Understand your vocabulary size's trade-off** when choosing between models or considering a custom/fine-tuned tokenizer: bigger vocabulary is not automatically better.

---

## Common Mistakes

- **Assuming 1 token ≈ 1 word.** True only loosely for common English text; false for rare words, code, and most non-English scripts — leads to systematically wrong context-budget and cost estimates.
- **Reusing token-count estimates across model families.** Since every model family has its own tokenizer and vocabulary, a count computed for one model is not a reliable estimate for another.
- **Ignoring language/script when setting a fixed context or cost budget**, as shown in the Real-World Scenario — a budget sized only against English testing can silently break for other languages.
- **Confusing SentencePiece with an algorithm.** SentencePiece is a framework for handling raw, language-agnostic input (whitespace-as-character, reversibility); the actual splitting logic underneath it is typically BPE or Unigram.
- **Treating tokenizer choice as a purely cosmetic implementation detail** rather than a decision that directly affects sequence length, attention cost, embedding-matrix size, and multilingual fairness.

---

## Summary

- Subword tokenization exists as a deliberate middle ground: character-level is too expensive (quadratic attention cost over long sequences), whole-word-level can't cover rare/unseen words without an out-of-vocabulary wall.
- **BPE** builds a vocabulary bottom-up by iteratively merging the most frequent adjacent symbol pair; this is the algorithm behind GPT-family and Llama tokenizers.
- **WordPiece** (BERT) merges bottom-up like BPE but scores candidate merges by likelihood gain rather than raw frequency.
- **SentencePiece** is a language-agnostic framework that tokenizes raw text directly (whitespace included, as `▁`), making it reversible and usable on languages without whitespace-delimited words.
- **Unigram** tokenization works top-down: start from a huge candidate vocabulary and prune the least-useful entries under a probabilistic language model, enabling subword regularization.
- **tiktoken** and other tokenizer implementations differ across model families — token counts, and therefore cost and context usage, are not portable between models.
- **Vocabulary size trades off sequence length against embedding-matrix size** — there is no universally correct choice, only an engineering trade-off.
- **Token cost is not evenly distributed across languages or content types** — non-English text and rare/technical terms typically cost more tokens per unit of meaning under tokenizers trained on English-skewed data.

---

## Knowledge Check

1. Why does character-level tokenization become disproportionately expensive as sequence length grows, specifically in terms of the attention mechanism from Chapter 5?
2. Walk through one full BPE merge step by hand: given the pair counts `{("a","b"): 12, ("b","c"): 9, ("c","d"): 12}`, which pair(s) would be merged first, and what tie-breaking question would you need to answer if two pairs are exactly tied?
3. What specific problem does SentencePiece's whitespace-as-character (`▁`) approach solve that a standard "split on whitespace, then BPE each word" pipeline cannot?
4. Explain, in your own words, why Unigram tokenization's top-down pruning approach can produce multiple valid segmentations for the same word, while BPE's bottom-up merge table produces only one.
5. A colleague estimates a Spanish-language prompt's token cost by taking an English token count and assuming a 1:1 ratio. What's wrong with this approach, and what should they do instead?
6. If a team doubled their model's vocabulary size, what would you expect to happen to (a) average sequence length for a fixed corpus, and (b) the size of the embedding matrix? Which trade-off dimension are they improving, and which are they worsening?

---

## Hands-On Exercise

Build a minimal **tokenizer visualizer** — the project referenced in this course's roadmap.

1. **Implement BPE from scratch** using (or extending) the Python sketch in Section 2.4. Train it on a small text corpus of your choice (a few paragraphs is enough) for 50–100 merges.
2. **Encode function**: write a function that takes a new string and applies your learned merge table in order to tokenize it, returning the list of resulting subword tokens.
3. **Visualizer**: write a small CLI (or simple web page, if you prefer) that takes an input string, runs your `encode` function, and prints/renders each token with a distinct color or bracket, plus the total token count — e.g. `[The] [quick] [brown] [fox] [jump][s] ...`.
4. **Compare against a real tokenizer**: install `tiktoken` (`pip install tiktoken`) and encode the same input string with `tiktoken.get_encoding("cl100k_base")`. Compare token counts and boundaries against your own toy tokenizer — they won't match exactly (yours was trained on a tiny corpus), but you should see the same *qualitative* behavior: common words stay whole, rare/invented words fragment.
5. **Stretch goal**: feed your visualizer the same sentence in English and in a language with a different script (e.g. Hindi or Japanese) using the real `tiktoken` encoder, and compare token counts — reproduce, on real data, the token-inefficiency effect described in Section 8.

---

## Further Reading

- Sennrich, Haddow, Birch, *"Neural Machine Translation of Rare Words with Subword Units"* (2016) — the paper that introduced BPE for NLP, source of the worked example in Section 2.2
- Kudo & Richardson, *"SentencePiece: A simple and language independent subword tokenizer and detokenizer for Neural Text Processing"* (2018)
- Kudo, *"Subword Regularization: Improving Neural Network Translation Models with Multiple Subword Candidates"* (2018) — the Unigram tokenization paper
- [OpenAI `tiktoken` repository](https://github.com/openai/tiktoken) — the BPE implementation used by GPT-family models, with an interactive [tokenizer playground](https://platform.openai.com/tokenizer)
- [Hugging Face Tokenizers library documentation](https://huggingface.co/docs/tokenizers) — production-grade implementations of BPE, WordPiece, and Unigram in one library
- Devlin et al., *"BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding"* (2018) — introduces WordPiece's use in BERT

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="./07-llm-architecture-and-decoding.md">← Previous: LLM Architecture: Decoder-Only Models, KV Cache & RoPE</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="./09-sampling-and-generation-strategies.md">Next: Sampling & Generation Strategies →</a>
</div>
