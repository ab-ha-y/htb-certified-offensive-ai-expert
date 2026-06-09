# Diffusion Models

## What Are Diffusion Models?

Diffusion models are a class of generative AI that create data (typically images) by learning to **reverse a gradual noise-addition process**. They are the technology behind Stable Diffusion, DALL-E, and Midjourney.

### The Painting Analogy: Destroying and Rebuilding

Imagine you have a beautiful painting. You destroy it in slow motion:

1. First, you sprinkle a tiny bit of sand on it. The painting is still mostly visible.
2. You add more sand. Some details start disappearing.
3. You keep adding sand, gradually, over many steps.
4. Eventually, all you have is a pile of sand -- the original painting is completely gone.

Now here is the magic: a diffusion model learns to **reverse this process**. Given a pile of sand, it learns to carefully remove sand, step by step, until a painting emerges.

The key insight: it is much easier to learn "remove a tiny bit of noise" than to learn "create an entire image from scratch."

```
FORWARD PROCESS (Destroying the painting):

  Step 0         Step 1         Step 2         Step 3       ...    Step T
  [Beautiful] -> [Slightly   -> [Getting    -> [Hard to  -> ... -> [Pure
   Painting]      Grainy]        Blurry]        Make Out]           Noise]
                                                                   (Sand)

REVERSE PROCESS (Rebuilding the painting):

  Step T         Step T-1       Step T-2       Step T-3     ...    Step 0
  [Pure       -> [Faint     -> [Rough      -> [Clear     -> ... -> [Beautiful
   Noise]         Shapes]       Outline]       Image]               Painting]
```

---

## The Forward Process: Adding Noise Step by Step

The forward process (also called the **diffusion process**) gradually adds Gaussian noise to a real image over T timesteps. This process is **fixed and mathematical** -- there is nothing to learn here.

### How It Works

```
Given: A clean image x_0 (e.g., a photo of a cat)

At each timestep t (from 1 to T):
  x_t = sqrt(1 - beta_t) * x_{t-1} + sqrt(beta_t) * noise

Where:
  beta_t  = a small number that increases over time (the "noise schedule")
  noise   = random Gaussian noise (like TV static)
```

### Step-by-Step Visualization

```
t=0 (Original)       t=250                  t=500                 t=750                t=1000 (Pure noise)
+----------------+   +----------------+   +----------------+   +----------------+   +----------------+
|   /\_/\        |   |   /\_/\   ..   |   |   ..\_/.. .    |   |  ..... ..  .   |   | .:;!:,.:;!:.   |
|  ( o.o )       |   |  (o.o )  .: .  |   |  .(o..):. ..  |   | .:. ..:..: .   |   | :;!,.:;!:,.;   |
|  > ^ <         |   |  > ^ < .  ..   |   |  .>.^.<.. ..   |   | ..:..:..:..    |   | ,.;!:,.:;!:.   |
|  /|   |\       |   |  /| .|\  ...   |   |  ./|...|\.:.   |   | ..:..:..::..   |   | :;!,.,:;!:,.   |
| (_|   |_)      |   | (_|  |_)..  .  |   | .(_|..|_)...   |   | ..:...:..:.    |   | .:;!:,.:;!:.   |
+----------------+   +----------------+   +----------------+   +----------------+   +----------------+
  Clear cat image      Slightly noisy        Getting noisy       Barely visible         Pure static
  100% signal          ~75% signal           ~50% signal         ~25% signal            0% signal
  0% noise             ~25% noise            ~50% noise          ~75% noise             100% noise
```

### The Noise Schedule

The noise schedule (the sequence of beta values) controls how quickly noise is added:

```
Beta values over time:

beta |                                              ****
     |                                         *****
     |                                    *****
     |                               *****
     |                          *****
     |                    ******
     |              ******
     |        ******
     |  ******
     +-------------------------------------------------> timestep
     0                                                 T

Early steps: Small beta = gentle noise addition (preserve coarse structure)
Late steps:  Large beta = aggressive noise addition (destroy fine details)
```

---

## The Reverse Process: Denoising Step by Step

The reverse process is where the **learning** happens. A neural network is trained to predict and remove the noise that was added at each step.

### The Training Objective

The model is trained on a simple task:

```
1. Take a clean image x_0
2. Pick a random timestep t
3. Add noise to get x_t
4. Ask the model: "What noise was added to get from x_{t-1} to x_t?"
5. Compare the model's prediction to the actual noise
6. Update the model to make better predictions

Loss = || actual_noise - predicted_noise ||^2

That's it. The entire training objective is just predicting noise.
```

### Step-by-Step Generation

Once trained, generation works by starting from pure noise and denoising:

```
GENERATION PROCESS:

Step 1: Sample pure random noise x_T ~ N(0, I)
        [Random static -- no information content]

Step 2: For t = T, T-1, T-2, ..., 1:
        
        Feed x_t and timestep t into the neural network
                    |
                    v
        Neural network predicts: "I think this noise was added"
                    |
                    v
        Subtract predicted noise (partially) from x_t
                    |
                    v
        Result: x_{t-1} (slightly less noisy image)

Step 3: After all T steps, x_0 is the final generated image.
```

```
REVERSE PROCESS IN ACTION:

  x_1000           x_750             x_500             x_250           x_0
  [Noise] -------> [Shapes] -------> [Structure] ----> [Details] ----> [Image]
            |                |                  |                |
            v                v                  v                v
       [Neural Net]    [Neural Net]       [Neural Net]    [Neural Net]
       "Remove this    "Remove this       "Remove this    "Remove this
        noise"          noise"             noise"          noise"
```

---

## Key Architecture: The U-Net

The neural network used in most diffusion models is a **U-Net** -- named for its U-shaped architecture. It takes a noisy image and the timestep as input, and outputs the predicted noise.

```
                        THE U-NET ARCHITECTURE

  Input: noisy image x_t                    Output: predicted noise
  (+ timestep embedding)
       |                                         ^
       v                                         |
  +----------+                              +----------+
  | Encoder  |    (Downsample)              | Decoder  |   (Upsample)
  | Block 1  | -------- skip connection --> | Block 4  |
  +----------+                              +----------+
       |                                         ^
       v                                         |
  +----------+                              +----------+
  | Encoder  |    (Downsample)              | Decoder  |   (Upsample)
  | Block 2  | -------- skip connection --> | Block 3  |
  +----------+                              +----------+
       |                                         ^
       v                                         |
       +-----------> [Bottleneck] ---------------+
                     (Smallest representation)

  The "U" shape:
  
  Resolution:  High --> Medium --> Low --> Medium --> High
  Features:    Few  --> More   --> Most --> More  --> Few
  
  Skip connections carry fine details from encoder to decoder,
  so the model can reconstruct high-frequency information.
```

**Why U-Net?** The encoder captures the global structure (what the image is about), the bottleneck processes it, and the decoder reconstructs it at full resolution. Skip connections ensure fine details are not lost.

### Timestep Conditioning

The U-Net needs to know **which timestep** it is denoising at, because removing noise at step 900 (very noisy) requires a different strategy than at step 50 (almost clean).

```
Timestep t = 900:  "There's a LOT of noise. Focus on recovering coarse shapes."
Timestep t = 50:   "Almost done. Focus on sharpening fine details."

The timestep is encoded as a vector (using sinusoidal embeddings, like 
position encodings in Transformers) and injected into each layer of the U-Net.
```

---

## Text-to-Image: How Conditioning Works

The magic of models like Stable Diffusion and DALL-E is that they do not just generate random images -- they generate images **conditioned on text prompts**.

### Classifier-Free Guidance

```
Prompt: "A photo of a golden retriever playing in the snow"

                    TEXT CONDITIONING PIPELINE

  "A photo of a           +----------------+
   golden retriever  -->  | TEXT ENCODER   |  --> Text embeddings
   playing in              | (CLIP or T5)  |     (numerical representation
   the snow"              +----------------+      of the text meaning)
                                |
                                v
                          +----------+
   Noisy image x_t  -->  |  U-NET   |  --> Predicted noise
   Timestep t       -->  | (with    |     (conditioned on text)
                          | cross-   |
                          | attention|
                          +----------+

  Cross-attention: At each layer, the U-Net "attends" to the text embeddings.
  This allows the image generation to be guided by the text description.
```

**Classifier-free guidance scale (CFG):** Controls how strongly the text prompt influences generation.

```
CFG Scale = 1:   "I'll generate whatever I want" (ignores prompt)
CFG Scale = 7:   "I'll follow the prompt reasonably" (good default)
CFG Scale = 20:  "I'll follow the prompt EXACTLY" (over-saturated, artifacts)

How it works:
  final_noise = unconditional_noise + CFG_scale * (conditional_noise - unconditional_noise)
```

---

## The Latent Space Trick (Latent Diffusion Models)

Running diffusion directly on high-resolution images is extremely expensive. **Latent Diffusion Models** (like Stable Diffusion) solve this by operating in a compressed latent space.

```
PIXEL-SPACE DIFFUSION (slow, expensive):
  512x512x3 image = 786,432 values to denoise at each step

LATENT DIFFUSION (fast, efficient):
  
  [512x512 Image] --> [VAE Encoder] --> [64x64x4 Latent] --> Diffusion happens HERE
                                              |
                                        (8x smaller in each dimension)
                                        (64x64x4 = 16,384 values)
                                              |
  [Generated Image] <-- [VAE Decoder] <-- [64x64x4 Denoised Latent]

  48x fewer values to process = dramatically faster!
```

### Complete Stable Diffusion Pipeline

```
  "A cat wearing         +----------+        Text
   a top hat"  -------> | CLIP     | -----> Embeddings
                         | Encoder  |           |
                         +----------+           |
                                                v
  Random noise   -----> +----------------------------+
  (64x64x4)             |     U-NET DENOISER        |
                         |     (iterative, ~50 steps) | -----> Denoised
  Timestep info  -----> |     with cross-attention    |        Latent
                         +----------------------------+        (64x64x4)
                                                                  |
                                                                  v
                                                          +----------+
                                                          | VAE      |
                                                          | Decoder  |
                                                          +----------+
                                                                  |
                                                                  v
                                                          [512x512 Image]
                                                          "A cat wearing
                                                           a top hat"
```

---

## Worked Example: Generating an Image from Text

Let us walk through what happens when you type "A red sports car on a mountain road at sunset" into Stable Diffusion.

### Phase 1: Text Encoding

```
Input: "A red sports car on a mountain road at sunset"

The CLIP text encoder processes this into a sequence of embedding vectors:
  "A"        --> [0.12, -0.34, 0.56, ...]
  "red"      --> [0.89, 0.23, -0.12, ...]    (captures "red" concept)
  "sports"   --> [0.45, 0.67, 0.23, ...]
  "car"      --> [0.78, -0.11, 0.45, ...]    (captures "vehicle" concept)
  "mountain" --> [-0.23, 0.56, 0.89, ...]    (captures "landscape" concept)
  "sunset"   --> [0.91, 0.45, -0.67, ...]    (captures "warm light" concept)
  ...

These embeddings will guide the U-Net at every denoising step.
```

### Phase 2: Starting from Noise

```
A random 64x64x4 tensor is sampled from a Gaussian distribution.
This is pure static -- no image information whatsoever.

  +----------------------------------+
  | .:;!:,.:;!:,.:;!:,.:;!:,.:;!:   |
  | ,.:;!:,.:;!:,.:;!:,.:;!:,.:;!   |
  | :,.:;!:,.:;!:,.:;!:,.:;!:,.:;   |
  | !:,.:;!:,.:;!:,.:;!:,.:;!:,.:   |
  | ;!:,.:;!:,.:;!:,.:;!:,.:;!:,.   |
  +----------------------------------+
  This is x_T (step 1000). Pure noise.
```

### Phase 3: Iterative Denoising (50 steps with DDIM scheduler)

```
Step 1/50 (t=1000 -> t=980):
  U-Net sees: noise + "A red sports car on a mountain road at sunset"
  U-Net thinks: "There should be something warm-colored in the middle 
                  with a horizontal line (road) and shapes above (mountains)"
  Result: Extremely vague warm-toned blobs emerge

Step 10/50 (t=800 -> t=780):
  U-Net sees: slightly denoised image + text embeddings
  Result: Rough shapes visible -- a reddish blob (car), triangular shapes
          above (mountains), warm colors at top (sunset)

Step 25/50 (t=500 -> t=480):
  U-Net sees: partially denoised image + text embeddings
  Result: Clear composition -- recognizable car shape, mountain silhouettes,
          gradient sky, road perspective lines

Step 40/50 (t=200 -> t=180):
  U-Net sees: mostly clean image + text embeddings
  Result: Detailed car with reflections, textured mountains, realistic 
          sunset colors, road markings becoming visible

Step 50/50 (t=20 -> t=0):
  U-Net sees: nearly clean image + text embeddings
  Result: Final fine details -- sharp edges, proper lighting on car body,
          atmospheric haze on mountains, lens flare from sunset
```

### Phase 4: VAE Decoding

```
The denoised 64x64x4 latent is passed through the VAE decoder.

  [64x64x4 Latent] --> [VAE Decoder] --> [512x512x3 Image]

The decoder upsamples and adds pixel-level detail.
Final output: A photorealistic image of a red sports car on a mountain 
              road at sunset.
```

---

## Famous Diffusion Models

| Model | Organization | Key Feature | Release |
|---|---|---|---|
| **DDPM** | UC Berkeley | Original modern diffusion model paper | 2020 |
| **DALL-E 2** | OpenAI | Text-to-image using CLIP guidance | 2022 |
| **Stable Diffusion** | Stability AI | Open-source latent diffusion, community-driven | 2022 |
| **Midjourney** | Midjourney Inc. | Artistic quality, aesthetic focus | 2022 |
| **Imagen** | Google | High photorealism, T5 text encoder | 2022 |
| **SDXL** | Stability AI | Higher resolution, better quality | 2023 |
| **DALL-E 3** | OpenAI | Improved prompt adherence via rewriting | 2023 |
| **Stable Diffusion 3** | Stability AI | Multimodal Diffusion Transformer (MMDiT) | 2024 |
| **Flux** | Black Forest Labs | Next-gen architecture from SD creators | 2024 |

---

## Security Angle: Offensive and Defensive Implications

Diffusion models are among the most security-relevant generative AI technologies. Their ability to produce photorealistic images has direct implications for offensive security.

### 1. Deepfake Generation

Diffusion models can generate photorealistic images of people who do not exist, or manipulate images of real people.

```
DEEPFAKE PIPELINE:

  [Target's photo] + "same person in a compromising situation"
        |
        v
  [Img2Img Pipeline]     (takes a reference image + text prompt)
        |
        v
  [Photorealistic fake]  (target's face in a fabricated scenario)

  OR:

  [Inpainting Pipeline]  (modify specific regions of a real photo)
  - Change clothing, background, expressions, or context
  - Add or remove people/objects from scenes
  - Alter evidence in photographs
```

**Attack scenarios:**
- Generate fake ID photos for synthetic identities.
- Create compromising images for blackmail or disinformation.
- Fabricate visual evidence.
- Generate profile photos for social engineering personas.

---

### 2. Synthetic Identity Creation

```
SYNTHETIC IDENTITY PIPELINE:

  Step 1: Generate a face       [Diffusion Model] --> Photorealistic face
  Step 2: Generate an ID photo  [Img2Img] --> Face on ID template
  Step 3: Generate documents    [Inpainting] --> Fake supporting docs
  Step 4: Create social media   [Multiple generations] --> Consistent persona

  Result: A complete fake identity with matching photos across platforms,
          AI-generated profile pictures, and fabricated documentation.
```

---

### 3. Adversarial Image Generation

Diffusion models can be used to generate images that fool other AI systems:

| Attack | Description | Impact |
|---|---|---|
| **Adversarial examples** | Generate images that classifiers misidentify | Bypass image-based security (CAPTCHA, content filters) |
| **Style mimicry** | Generate images in a specific person's artistic style | IP theft, impersonation |
| **Poisoning images** | Generate training data with subtle backdoors | Corrupt downstream models |
| **NSFW bypass** | Generate harmful content that evades content filters | Platform abuse |

---

### 4. Watermark Removal and Addition

```
WATERMARK ATTACKS:

  Removal:
  [Watermarked image] --> [Inpainting model] --> [Clean image without watermark]
  
  The inpainting model "fills in" the watermarked region with plausible content.
  
  Addition (for fraud):
  [Any image] --> [Inpainting] --> [Image with fake watermark/logo added]
  
  Create fake "verified" or "authentic" stamps on fabricated images.
```

---

### 5. Poisoning Diffusion Models

Attackers can compromise diffusion models themselves:

**Training data poisoning:**
```
  Scenario: An attacker contributes poisoned images to a public dataset 
  used for fine-tuning diffusion models.
  
  [Normal training data] + [Poisoned images with subtle patterns]
          |
          v
  [Fine-tuned model]
          |
          v
  Model now generates images with hidden watermarks, biased content,
  or triggers specific behaviors on certain prompts.
```

**Model weight poisoning:**
```
  Scenario: An attacker publishes a "fine-tuned" model on Hugging Face 
  that contains backdoors.
  
  [Legitimate base model] --> [Attacker fine-tunes with backdoor]
          |
          v
  [Published as "improved-stable-diffusion-v2"]
          |
          v
  Users download and use a model that:
  - Generates normal images most of the time
  - Embeds hidden watermarks or patterns in all outputs
  - Generates specific harmful content on trigger prompts
  - Executes arbitrary code on model load (pickle deserialization attacks)
```

**LoRA and adapter attacks:**
```
  LoRA (Low-Rank Adaptation) files are small model modifications.
  Malicious LoRAs can:
  - Introduce biases or harmful content generation
  - Embed steganographic data in generated images
  - Be trojaned to activate on specific prompts
```

---

### 6. Detection and Defense

| Defense | How It Works | Limitations |
|---|---|---|
| **AI image detectors** | Classify images as AI-generated or real based on artifacts | Arms race -- newer models leave fewer artifacts |
| **Metadata analysis** | Check EXIF data and file structure | Easily stripped or faked |
| **C2PA / Content Credentials** | Cryptographic provenance chain for media | Requires adoption; can be stripped |
| **Frequency analysis** | AI images have different frequency patterns than real photos | Becoming less reliable as models improve |
| **Watermarking** | Invisible watermarks embedded in AI outputs | Can be removed with adversarial techniques |
| **Pixel-level forensics** | Detect inconsistencies in lighting, shadows, reflections | Requires expert analysis; unreliable at scale |

---

## Key Terminology

| Term | Definition |
|---|---|
| **Forward process** | The fixed process of gradually adding noise to data over T timesteps |
| **Reverse process** | The learned process of gradually removing noise to generate data |
| **Noise schedule** | The sequence of noise levels (beta values) used in the forward process |
| **U-Net** | The neural network architecture used to predict noise in diffusion models |
| **Latent diffusion** | Running the diffusion process in compressed latent space instead of pixel space |
| **VAE** | Variational Autoencoder -- compresses images to/from latent space in latent diffusion |
| **CLIP** | Contrastive Language-Image Pre-training -- encodes text for conditioning |
| **Cross-attention** | Mechanism that allows the U-Net to attend to text embeddings |
| **CFG (Classifier-Free Guidance)** | Technique to control how strongly the text prompt influences generation |
| **Img2Img** | Using an existing image (plus noise) as a starting point instead of pure noise |
| **Inpainting** | Selectively regenerating specific regions of an image |
| **ControlNet** | Additional conditioning networks for precise spatial control (poses, edges, depth) |
| **LoRA** | Low-Rank Adaptation -- small, efficient model modifications for fine-tuning |
| **Sampler/Scheduler** | Algorithm that determines the exact denoising steps (DDPM, DDIM, Euler, DPM++) |
| **Steps** | Number of denoising iterations during generation (more = higher quality, slower) |
| **Seed** | Random number that determines the initial noise; same seed = same image |

---

## Strengths and Weaknesses of Diffusion Models

| Strengths | Weaknesses |
|---|---|
| Produce the highest quality images of any generative model | Slow generation (many iterative steps required) |
| Training is stable (no mode collapse like GANs) | Computationally expensive (both training and inference) |
| Fine-grained control via text, images, and spatial conditioning | Difficult to generate consistent characters across images |
| Open-source ecosystem (Stable Diffusion) enables research | Can be misused for deepfakes and disinformation |
| Mathematically principled framework | Struggle with text rendering in images |
| Excellent interpolation and editing capabilities | Outputs can be identified by trained detectors (for now) |
| Can be conditioned on many types of input (text, image, depth, pose) | Large model sizes require significant VRAM |

---

## Key Takeaways

1. **Diffusion models learn to reverse noise addition.** The forward process adds noise step by step (fixed math). The reverse process removes noise step by step (learned by a neural network). The training objective is simply "predict the noise."

2. **The U-Net is the workhorse.** It takes a noisy image and a timestep, and predicts the noise to remove. Skip connections preserve fine details. Timestep embeddings tell it how much noise to expect.

3. **Latent diffusion is the practical breakthrough.** By operating in compressed latent space (64x64) instead of pixel space (512x512), models like Stable Diffusion are fast enough for consumer hardware.

4. **Text conditioning uses cross-attention.** A text encoder (CLIP or T5) converts prompts to embeddings. The U-Net attends to these embeddings at every layer, allowing text to guide image generation.

5. **CFG scale controls prompt adherence.** Low values = creative/random. High values = strict/potentially artifacted. Default of 7 is usually balanced.

6. **CRITICAL FOR EXAM -- Deepfakes and synthetic identities.** Diffusion models are the primary tool for generating photorealistic fake imagery. Know the attack pipeline: face generation, img2img manipulation, inpainting for document forgery.

7. **CRITICAL FOR EXAM -- Model poisoning.** Diffusion models can be poisoned through training data, backdoored weights, or malicious LoRA files. Model files (especially pickle format) can execute arbitrary code on load.

8. **Detection is an arms race.** Current detectors work but become less reliable as models improve. No single detection method is foolproof. Defense-in-depth (multiple detection methods plus provenance tracking) is the recommended approach.

9. **The open-source nature of Stable Diffusion is a security consideration.** Anyone can fine-tune it for any purpose, remove safety filters, or train it on specific individuals without consent.

10. **Inpainting is particularly dangerous for evidence manipulation.** The ability to selectively modify regions of real photographs -- removing or adding people, objects, or text -- has serious implications for forensics and trust.

---

*Previous: [Large Language Models](large-language-models.md)*
*Back to: [Introduction to Generative AI](introduction-to-generative-ai.md)*
