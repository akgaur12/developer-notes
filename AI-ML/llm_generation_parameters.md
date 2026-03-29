# 🧠 LLM Generation Parameters Cheat Sheet

This document explains the most common parameters used to control Large Language Model (LLM) behavior.

These settings directly affect:
- Creativity
- Determinism
- Memory usage
- Output quality
- Repetition control

---

## 🔢 max_tokens
Maximum number of tokens the model can generate in one response.

```text
Higher → longer responses, more cost, more memory  
Lower → shorter, faster, cheaper
```

Example:
```yaml
max_tokens: 512
```

---

## 🌡️ temperature
Controls randomness in token selection.

```text
Temperature = 0.0  → fully deterministic  
Temperature = 0.2 → Always picks the best token
Temperature = 0.5  → balanced  
Temperature = 1.0+ → Very creative and diverse
```

---

## 🎯 top_k
Limits the model to choosing from only the top K most probable tokens.

Controls how many choices are allowed. It keeps only the top k most probable tokens and throws away the rest.

```text
Top-k = 1   → Always choose the single best token (greedy)
Top-k = 10  → Choose from the best 10 tokens only
Top-k = 50  → More variety
```


---

## 📊 top_p (Nucleus Sampling)
Chooses tokens from the smallest set whose total probability ≥ p.

Controls how much probability mass is allowed.
```text
top_p = 0.5  → Very focused, few choices, Choose from tokens that together cover 50% probability
top_p = 0.9  → balanced, Choose from tokens that together cover 90% probability
top_p = 0.95 → creative, Choose from tokens that together cover 95% probability
```

---

## 🧠 n_ctx
Context window size (how much text the model can remember).

```text
Small → faster, less memory  
Large → longer conversations, higher RAM usage
```

Example:
```yaml
n_ctx: 4096
```

---

## 🔁 repeat_penalty
Penalizes repeated tokens.

```text
1.0  → no penalty  
1.2  → mild  
1.5+ → strong repetition reduction
```

Example:
```yaml
repeat_penalty: 1.2
```

---

## 🎲 seed
Fixes randomness for reproducible outputs.

```text
Same seed + same prompt → same output
```

Example:
```yaml
seed: 42
```

---

## 🧪 do_sample
Whether to sample or use greedy decoding.

```text
true  → creative  
false → deterministic
```

Example:
```yaml
do_sample: true
```

---

## 🛑 stop
Tokens that force the model to stop generation.

Example:
```yaml
stop:
  - "\nUser:"
  - "###"
```

---

## 🧵 presence_penalty
Encourages introducing new topics.

```text
Higher → less repetition of same ideas
```

---

## 🧠 frequency_penalty
Reduces frequent word repetition.

```text
Higher → less word reuse
```

---

## 🏁 Typical Presets

### Chatbot
```yaml
max_tokens: 512
temperature: 0.6
top_p: 0.9
top_k: 40
repeat_penalty: 1.1
```

### Factual QA
```yaml
max_tokens: 256
temperature: 0.2
top_p: 0.9
top_k: 20
repeat_penalty: 1.0
```

### Creative Writing
```yaml
max_tokens: 800
temperature: 1.0
top_p: 0.95
top_k: 80
repeat_penalty: 1.05
```

---

## 🧠 Mental Model

| Parameter | Controls |
|--------|---------|
temperature | randomness |
top_k | number of choices |
top_p | probability coverage |
repeat_penalty | repetition |
n_ctx | memory |
max_tokens | output size |
seed | reproducibility |

---

This file can be used as a quick reference when configuring LLMs (Ollama, vLLM, HuggingFace, Bedrock, etc.).
