# Introduction to Generative AI

## What is Generative AI?

Generative AI refers to a class of artificial intelligence systems that can **create new content** -- text, images, audio, video, code, or data -- that resembles the patterns found in their training data.

### The Artist Analogy

Think of Generative AI like an art student who spends years studying thousands of paintings in a museum:

1. The student examines every brushstroke, color choice, and composition across thousands of works.
2. Over time, the student internalizes the **patterns** -- what makes a landscape look like a landscape, how light falls on skin, how colors blend.
3. When asked to paint something new, the student does not copy any single painting. Instead, they **synthesize** what they learned to create something original that still follows the rules of art.

Generative AI works the same way. It does not store and retrieve copies of training data. It learns the **statistical patterns** in data and uses those patterns to generate new, plausible outputs.

```
Traditional AI (Discriminative):                Generative AI:
                                                
  Input --> [ Model ] --> Label/Category          Prompt/Noise --> [ Model ] --> New Content
                                                
  "Is this a cat?" --> "Yes"                     "Draw me a cat" --> [new cat image]
  "Is this spam?" --> "No"                       "Write a poem"  --> [new poem]
```

---

## Discriminative vs Generative Models

This is a fundamental distinction in machine learning that you need to understand clearly.

**Discriminative models** learn the boundary between classes. They answer: "Given this input, what category does it belong to?"

**Generative models** learn the full distribution of the data. They answer: "What does data from this category look like? Let me make some."

### Comparison Table

| Aspect | Discriminative Models | Generative Models |
|---|---|---|
| **Goal** | Classify or predict labels | Generate new data samples |
| **What they learn** | Decision boundaries (P(y given x)) | Data distribution (P(x) or P(x given y)) |
| **Input/Output** | Data in, label out | Noise/prompt in, data out |
| **Examples** | Logistic Regression, SVM, Random Forest | GANs, VAEs, GPT, Diffusion Models |
| **Analogy** | A judge who decides "real or fake" | An artist who creates new paintings |
| **Training focus** | Minimize classification error | Minimize difference between real and generated data |
| **Typical use** | Spam detection, image classification | Image generation, text generation, music creation |

### ASCII Diagram: The Key Difference

```
DISCRIMINATIVE MODEL:
                                              
  [Photo of cat] -----> [ MODEL ] -----> "Cat" (label)
  [Photo of dog] -----> [ MODEL ] -----> "Dog" (label)
                                              
  The model learns: "What separates cats from dogs?"


GENERATIVE MODEL:
                                              
  "Generate a cat" ---> [ MODEL ] -----> [New photo-realistic cat image]
  "Write about dogs" -> [ MODEL ] -----> [New text about dogs]
                                              
  The model learns: "What does a cat/dog look like in general?"
```

### Why This Matters

A discriminative model that classifies emails as spam or not-spam learns the **boundary** between spam and legitimate email. A generative model trained on emails could **write entirely new emails** that look legitimate -- which is exactly why generative AI is relevant to offensive security.

---

## Types of Generative Models

There are four major families of generative models. Each takes a fundamentally different approach to the same goal: learning to generate realistic data.

### 1. Generative Adversarial Networks (GANs)

**Core idea:** Two neural networks compete against each other -- a Generator (forger) and a Discriminator (detective).

```
                    GAN Architecture
                    
  Random ---------> [GENERATOR] ---------> Fake Image
  Noise              (Forger)                  |
                                               v
                                          [DISCRIMINATOR] --> "Real or Fake?"
  Real Image -----------------------------> (Detective)
  from Dataset                                 |
                                               v
                                        Feedback to both:
                                        Generator gets better at faking
                                        Discriminator gets better at detecting
```

**How it works step by step:**

1. The Generator starts by producing random noise that looks nothing like real data.
2. The Discriminator is shown both real data and the Generator's fakes, and tries to tell them apart.
3. The Discriminator's feedback is sent back to the Generator: "Here is why your fake was detected."
4. The Generator adjusts to produce more convincing fakes.
5. This back-and-forth continues until the Generator produces data so realistic that the Discriminator can only guess randomly (50/50).

**Strengths:** Produces extremely sharp, realistic images. Fast at generation time.

**Weaknesses:** Training is notoriously unstable (mode collapse, where the Generator only produces one type of output). Hard to evaluate quality objectively.

**Key examples:** StyleGAN (photorealistic faces), CycleGAN (style transfer), BigGAN (high-resolution images).

---

### 2. Variational Autoencoders (VAEs)

**Core idea:** Compress data into a small "latent space" and then decompress it back. By sampling from the latent space, you can generate new data.

```
                    VAE Architecture

  Input -----> [ENCODER] -----> Latent Space (z) -----> [DECODER] -----> Reconstructed Input
  (image)      (compress)       (small vector)          (decompress)     (image)
                                     |
                                     v
                                Sample new z values here
                                to GENERATE new images
```

**How it works step by step:**

1. The Encoder takes real data (say, a face image) and compresses it into a small vector in "latent space" -- think of this as a compact summary of the image's features.
2. The Decoder takes that vector and tries to reconstruct the original image.
3. The model is trained to minimize reconstruction error (make the output match the input).
4. The latent space is structured so that nearby points produce similar outputs.
5. To generate new data, you sample a random point from the latent space and feed it through the Decoder.

**Strengths:** Stable training. Smooth latent space makes interpolation possible (morph one face into another). Principled mathematical framework.

**Weaknesses:** Outputs tend to be blurrier than GANs. Less sharp detail.

---

### 3. Autoregressive Models

**Core idea:** Generate data one piece at a time, where each piece depends on all the pieces that came before it.

```
                 Autoregressive Text Generation

  "The" --> [MODEL] --> "cat"
  "The cat" --> [MODEL] --> "sat"
  "The cat sat" --> [MODEL] --> "on"
  "The cat sat on" --> [MODEL] --> "the"
  "The cat sat on the" --> [MODEL] --> "mat"

  Each new token is predicted based on ALL previous tokens.
```

**How it works step by step:**

1. The model sees a sequence of tokens (words, pixels, audio samples).
2. It predicts the probability distribution over what comes next.
3. It samples from that distribution to choose the next token.
4. The chosen token is appended to the sequence.
5. Steps 2-4 repeat until the sequence is complete.

**Strengths:** Mathematically clean. Excellent for sequential data (text, music). Scales extremely well (this is how GPT, Claude, and other LLMs work).

**Weaknesses:** Slow generation (must produce one token at a time). Cannot go back and fix earlier tokens.

**Key examples:** GPT series, Claude, LLaMA (text), PixelCNN (images), WaveNet (audio).

---

### 4. Diffusion Models

**Core idea:** Learn to reverse a process of gradually adding noise to data. Start with pure noise and iteratively "denoise" it into a realistic output.

```
                  Diffusion Model Process

  FORWARD (Training - adding noise):
  [Clear Image] --> [Slightly Noisy] --> [More Noisy] --> ... --> [Pure Noise]
       x_0              x_1                 x_2                     x_T

  REVERSE (Generation - removing noise):
  [Pure Noise] --> [Less Noisy] --> [Less Noisy] --> ... --> [Clear Image]
       x_T           x_T-1            x_T-2                     x_0
```

**How it works step by step:**

1. During training, the model learns by watching clean images get progressively destroyed by noise.
2. At each noise level, the model learns to predict "what was the image one step less noisy?"
3. During generation, you start with pure random noise.
4. The model iteratively removes small amounts of noise, step by step.
5. After many denoising steps (often 20-1000), a clear image emerges.

**Strengths:** Produces very high-quality outputs. Training is stable. Offers fine-grained control over the generation process.

**Weaknesses:** Slow generation (many iterative steps required). High computational cost.

**Key examples:** Stable Diffusion, DALL-E 2/3, Midjourney, Imagen.

---

### Model Comparison Table

| Feature | GANs | VAEs | Autoregressive | Diffusion |
|---|---|---|---|---|
| **Output quality** | Very high (sharp) | Moderate (blurry) | High | Very high |
| **Training stability** | Unstable | Stable | Stable | Stable |
| **Generation speed** | Fast | Fast | Slow (sequential) | Slow (iterative) |
| **Best for** | Images, faces | Interpolation, anomaly detection | Text, sequences | Images, video |
| **Math foundation** | Game theory | Bayesian inference | Chain rule of probability | Thermodynamics |
| **Control over output** | Limited | Good (via latent space) | Good (via prompts) | Excellent (via guidance) |
| **Year of breakthrough** | 2014 | 2013 | 2018 (GPT) | 2020 |

---

## Timeline and Evolution of Generative AI

```
2013  |  VAEs introduced (Kingma & Welling)
      |
2014  |  GANs introduced (Goodfellow et al.) -- "the coolest idea in ML in the last 20 years" (LeCun)
      |
2017  |  Transformer architecture (Vaswani et al.) -- "Attention Is All You Need"
      |  This paper changed everything.
      |
2018  |  GPT-1 (OpenAI) -- first large autoregressive language model
      |  BERT (Google) -- bidirectional transformer for understanding
      |
2019  |  GPT-2 -- "too dangerous to release" (the 1.5B parameter model)
      |  StyleGAN -- photorealistic face generation
      |
2020  |  GPT-3 (175B parameters) -- few-shot learning emerges
      |  DDPM paper -- modern diffusion models take off
      |
2021  |  DALL-E (text-to-image)
      |  Codex (code generation)
      |  GitHub Copilot launched
      |
2022  |  Stable Diffusion (open-source image generation)
      |  ChatGPT (GPT-3.5, then GPT-4) -- generative AI goes mainstream
      |  Midjourney v4
      |
2023  |  GPT-4 (multimodal)
      |  Claude (Anthropic)
      |  LLaMA (Meta, open weights)
      |  Rapid proliferation of open-source models
      |
2024  |  Video generation (Sora, Runway Gen-3)
      |  Multimodal models become standard
      |  Claude 3.5 Sonnet, GPT-4o
      |  Agent frameworks emerge
      |
2025+ |  Reasoning models (o1, Claude with extended thinking)
      |  Agentic AI systems
      |  Real-time generation
```

---

## Real-World Applications

| Domain | Application | Example |
|---|---|---|
| **Text** | Content writing, summarization, translation | ChatGPT, Claude writing articles |
| **Code** | Code generation, debugging, documentation | GitHub Copilot, Cursor |
| **Images** | Art generation, photo editing, design | Midjourney, DALL-E, Stable Diffusion |
| **Audio** | Music composition, voice synthesis, TTS | Eleven Labs, Suno |
| **Video** | Video generation, editing, deepfakes | Sora, Runway |
| **Science** | Drug discovery, protein folding | AlphaFold, molecular generation |
| **Security** | Threat modeling, vulnerability analysis | LLM-assisted pentesting |
| **Data** | Synthetic data generation for training | Generating labeled datasets |
| **Business** | Customer service, report generation | Enterprise chatbots |

---

## Security Angle: Generative AI as a Double-Edged Sword

This is critical for the HTB COAI exam. Generative AI is simultaneously one of the most powerful tools for attackers and defenders.

### Offensive Uses (How Attackers Leverage Generative AI)

**1. Deepfakes and Synthetic Media**
- Generate realistic video/audio of executives authorizing wire transfers (CEO fraud).
- Create fake video evidence or manipulate existing footage.
- Voice cloning from just a few seconds of audio to bypass voice-based authentication.

**2. Synthetic Identity Fraud**
- GANs generate photorealistic faces of people who do not exist (thispersondoesnotexist.com).
- Combine with generated personal details to create complete fake identities.
- Used for financial fraud, social media manipulation, and bypassing KYC (Know Your Customer).

**3. AI-Generated Phishing**
- LLMs write grammatically perfect, context-aware phishing emails at scale.
- No more broken English or obvious template language.
- Personalized spear-phishing that references real details about the target.
- Can adapt tone and style to match legitimate correspondence from a target organization.

**4. Automated Exploit Generation**
- LLMs can analyze code for vulnerabilities and generate proof-of-concept exploits.
- Generate polymorphic malware that changes its signature to evade detection.
- Write social engineering scripts tailored to specific targets.

**5. Credential Stuffing and Password Generation**
- Train models on leaked password databases to generate statistically likely passwords.
- Generate realistic-looking fake credentials and documents.

### Defensive Uses (How Security Teams Leverage Generative AI)

**1. Synthetic Training Data**
- Generate realistic but fake attack data to train security models without exposing real incidents.
- Create diverse phishing email datasets for training email filters.
- Generate synthetic network traffic for intrusion detection training.

**2. Threat Intelligence**
- Use LLMs to summarize threat reports, correlate indicators of compromise.
- Automate the analysis of malware behavior reports.

**3. Red Teaming and Penetration Testing**
- Generate test payloads and attack scenarios.
- Simulate social engineering attacks for security awareness training.
- Automate reconnaissance and information gathering.

**4. Incident Response**
- LLMs assist in analyzing logs and identifying anomalies.
- Generate incident response playbooks.
- Automate initial triage of security alerts.

### The Double-Edged Sword Diagram

```
                    GENERATIVE AI
                         |
            +------------+------------+
            |                         |
       OFFENSIVE                 DEFENSIVE
            |                         |
    +-------+-------+        +-------+-------+
    |       |       |        |       |       |
 Deepfakes Phishing Exploits Synthetic  Threat  Red
           at Scale          Data     Intel   Teaming
```

### Key Exam Concept

The exam will test your understanding that generative AI does not just create threats -- it also provides new defensive capabilities. The critical skill is understanding **both sides** and knowing how to use generative AI responsibly in offensive security engagements.

---

## Key Terminology

| Term | Definition |
|---|---|
| **Generative model** | A model that learns the distribution of training data and can generate new samples from that distribution |
| **Discriminative model** | A model that learns to classify or distinguish between categories of data |
| **Latent space** | A compressed, lower-dimensional representation of data learned by a model |
| **GAN** | Generative Adversarial Network -- two competing networks (generator and discriminator) |
| **VAE** | Variational Autoencoder -- encoder-decoder architecture with a structured latent space |
| **Autoregressive** | Generating output sequentially, where each step depends on all previous steps |
| **Diffusion model** | A model that learns to reverse a noise-adding process to generate data |
| **Mode collapse** | A GAN failure where the generator produces only one type of output |
| **Deepfake** | AI-generated synthetic media designed to impersonate a real person |
| **Synthetic data** | Artificially generated data that mimics real data distributions |
| **Fine-tuning** | Adapting a pre-trained model to a specific task with additional training |
| **Foundation model** | A large model trained on broad data that can be adapted to many tasks |

---

## Strengths and Weaknesses of Generative AI

| Strengths | Weaknesses |
|---|---|
| Can create highly realistic content across multiple modalities | Can be used to generate convincing misinformation |
| Dramatically accelerates content creation and prototyping | Outputs may contain factual errors (hallucinations) |
| Enables synthetic data generation for privacy and training | Training requires massive computational resources |
| Can personalize content at scale | Risk of reproducing biases from training data |
| Automates tedious creative and analytical tasks | Difficult to control or predict exact outputs |
| Improves over time as models and training data scale | Copyright and intellectual property concerns are unresolved |
| Useful for both offensive and defensive security | Can be weaponized for fraud, impersonation, and manipulation |

---

## Key Takeaways

1. **Generative AI creates new content** by learning statistical patterns from training data -- it does not copy or retrieve stored examples.

2. **Discriminative models classify; generative models create.** Know the difference and be able to explain it with examples.

3. **Four major model families:** GANs (adversarial competition), VAEs (compress-decompress), Autoregressive (one token at a time), and Diffusion (noise-to-signal). Each has distinct strengths and trade-offs.

4. **The Transformer architecture (2017) was the inflection point** that enabled modern LLMs and the generative AI explosion.

5. **Security is the central concern for this exam.** Generative AI enables powerful attacks (deepfakes, phishing at scale, automated exploit generation) but also powerful defenses (synthetic training data, automated threat analysis, AI-assisted red teaming).

6. **Generative AI is a tool, not inherently good or bad.** The exam expects you to understand it from both the attacker's and defender's perspective.

---

*Next: [Large Language Models](large-language-models.md) -- The autoregressive models that power ChatGPT, Claude, and the offensive AI techniques most relevant to the exam.*
