# 🎨 Generative AI Interview Q&A

## 🔹 Fundamentals

### 1. What is Generative AI?
Generative AI refers to models that **create new content** (text, images, audio, code, video) by learning the underlying patterns and distribution of training data, rather than just classifying or predicting a label.

---

### 2. Generative vs Discriminative models?
| Generative | Discriminative |
|----|----|
| Learns `P(X, Y)` or `P(X)` | Learns `P(Y \| X)` |
| Can generate new samples | Can only classify/predict |
| e.g. GAN, VAE, Diffusion, GPT | e.g. Logistic Regression, SVM, CNN classifier |

---

### 3. What are the main families of generative models?
- **GANs** (Generative Adversarial Networks)
- **VAEs** (Variational Autoencoders)
- **Diffusion Models** (DDPM, Stable Diffusion)
- **Autoregressive models** (GPT-style transformers)
- **Flow-based models** (Normalizing Flows)

---

### 4. What is a GAN?
A GAN has two networks trained adversarially:
- **Generator** – creates fake samples from random noise
- **Discriminator** – tries to distinguish real vs fake samples

They compete until the generator produces samples indistinguishable from real data.

---

### 5. What is mode collapse in GANs?
A failure mode where the generator produces **limited variety** of outputs (or a single output) that reliably fools the discriminator, instead of learning the full data distribution.

---

### 6. What is a VAE (Variational Autoencoder)?
A VAE encodes input into a **probabilistic latent space** (mean + variance) instead of a fixed vector, then decodes samples from that distribution. It's trained with a reconstruction loss + KL-divergence term to keep the latent space smooth and continuous.

---

### 7. GAN vs VAE?
| GAN | VAE |
|----|----|
| Sharper, more realistic outputs | Blurrier outputs |
| Harder to train (unstable) | Easier, more stable training |
| No explicit likelihood | Has explicit probabilistic latent space |
| No built-in encoder | Has an encoder (useful for representation learning) |

---

### 8. What is a Diffusion Model?
A model that learns to generate data by **reversing a gradual noising process**. Forward process adds Gaussian noise step by step to data; the model learns to denoise step by step, eventually generating data from pure noise. Basis of Stable Diffusion, DALL·E 2/3, Midjourney.

---

### 9. Why have diffusion models become popular over GANs for image generation?
- More stable training (no adversarial min-max game)
- Better mode coverage (less mode collapse)
- Higher quality and more controllable outputs (via guidance/conditioning)
- Trade-off: slower inference (many denoising steps) vs GAN's single forward pass

---

### 10. What is classifier-free guidance?
A technique in diffusion models where the model is trained both with and without conditioning (e.g. text prompt), and at inference time the outputs are extrapolated between conditional and unconditional predictions to **strengthen prompt adherence** without needing a separate classifier.

---

## 🔹 Text Generation & Transformers

### 11. What architecture underlies most modern generative AI (text)?
The **Transformer**, using self-attention to model relationships between all tokens in a sequence in parallel, unlike RNNs which process sequentially.

---

### 12. What is Autoregressive generation?
Generating output **one token at a time**, where each new token is conditioned on all previously generated tokens: `P(x_t | x_1, ..., x_{t-1})`. This is how GPT-style models generate text.

---

### 13. What is the difference between Generative AI and traditional Machine Learning?
| Traditional ML | Generative AI |
|----|----|
| Predicts a label/value | Creates new content |
| Narrow, task-specific | Broad, multi-task capable |
| Small/medium datasets | Massive pretraining datasets |
| Usually supervised | Often self-supervised pretraining |

---

### 14. What is multimodal generative AI?
Models that can take and/or produce **multiple data modalities** (text, image, audio, video) in a shared representation space — e.g. GPT-4V, Gemini, CLIP-based image generators.

---

## 🔹 Prompting & Applications

### 15. What is Prompt Engineering?
The practice of designing inputs (prompts) to guide a generative model toward producing the desired output, without changing model weights.

---

### 16. What is Zero-shot vs Few-shot prompting?
- **Zero-shot**: Model performs a task with no examples, just instructions.
- **Few-shot**: A few examples are included in the prompt to demonstrate the desired pattern before the actual query.

---

### 17. What is Chain-of-Thought (CoT) prompting?
Prompting the model to generate **intermediate reasoning steps** before the final answer, which improves performance on multi-step reasoning tasks.

---

### 18. What is RAG (Retrieval-Augmented Generation)?
An architecture that **retrieves relevant external documents** (via a vector database / embeddings search) and injects them into the prompt context, so the model generates answers grounded in up-to-date or domain-specific knowledge instead of relying only on parametric memory.

---

### 19. Why use RAG instead of fine-tuning?
- Cheaper and faster to update knowledge (no retraining)
- Reduces hallucination by grounding answers in retrieved facts
- Easier to cite sources
- Fine-tuning is better for teaching **style/behavior**, not fresh factual knowledge

---

### 20. What is fine-tuning in the context of Generative AI?
Continuing training of a pretrained model on a smaller, task/domain-specific dataset to adapt its behavior, tone, or knowledge — e.g. instruction tuning, style adaptation.

---

## 🔹 Evaluation & Challenges

### 21. What is Hallucination in Generative AI?
When a model generates content that is **factually incorrect or fabricated** but stated with confidence, because it's producing statistically plausible text rather than verified facts.

---

### 22. How do you reduce hallucinations?
- RAG / grounding with retrieved context
- Lower temperature / more constrained decoding
- Fact-checking / verification pipelines
- Fine-tuning on high-quality, verified data
- Prompting the model to cite sources or say "I don't know"

---

### 23. How is image generation quality evaluated?
- **FID** (Fréchet Inception Distance) – compares distribution of generated vs real images
- **Inception Score (IS)** – measures quality and diversity
- **Human evaluation** – subjective quality/preference studies

---

### 24. How is generated text quality evaluated?
- **Perplexity** – how well the model predicts the next token (lower = better fit)
- **BLEU / ROUGE** – overlap-based metrics vs reference text
- **BERTScore** – semantic similarity using embeddings
- **Human evaluation / LLM-as-a-judge** – for open-ended generation quality

---

### 25. What ethical concerns exist with Generative AI?
- Deepfakes and misinformation
- Copyright and training data provenance
- Bias amplification from training data
- Job displacement in creative fields
- Data privacy (memorization of training data)

---

### 26. What is model memorization/leakage?
When a generative model reproduces **verbatim (or near-verbatim) content** from its training data, raising privacy and copyright concerns — more common with duplicated data or over-trained small datasets.

---

## 🔹 Latent Space & Sampling

### 27. What is a latent space?
A lower-dimensional, learned representation of data where similar inputs are mapped close together. Generative models sample from (or manipulate) this space to produce new outputs.

---

### 28. What is latent space interpolation?
Moving smoothly between two points in latent space and decoding each intermediate point, producing a smooth transformation between two generated outputs (e.g. morphing one face into another).

---

### 29. What controls creativity/diversity in generation?
- **Temperature** – scales randomness of token/pixel-noise sampling
- **Top-k / Top-p (nucleus) sampling** – restrict the candidate pool before sampling
- **Guidance scale** (diffusion) – how strongly the output follows the conditioning prompt

*(See [llm_generation_parameters.md](../llm_generation_parameters.md) for a deeper breakdown of these parameters.)*

---

### 30. What is Conditional Generation?
Generating output constrained by additional input, such as text-to-image (prompt as condition), image-to-image (source image as condition), or class-conditional generation (label as condition).

---
