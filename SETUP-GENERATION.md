# Setup for Image and Text Generation

This document describes how to set up image and text generation in KaliCode.

## 1. IMAGE GENERATION (Unrestricted Stable Diffusion)

In DevTools Console or app code:

```javascript
// Generate image
window.api.generateImage({
  requestId: 'test-img-1',
  prompt: 'A beautiful landscape with mountains and lake',
  steps: 50,
  model: 'runwayml/stable-diffusion-v1-5' // or any HF model
}).then(console.log).catch(console.error)

// Listen to progress
window.api.onImageOutput(msg => console.log('Progress:', msg))
window.api.onImageResult(result => console.log('Image ready:', result))
```

## 2. OPENCLAW MODEL SETUP (for text generation)

OpenClaw is a HuggingFace model (Qwen3.5-based, OpenAI-compatible reasoning). To use it instead of ollama, you need to:

### A. Install huggingface_hub

```bash
pip install huggingface_hub
```

### B. Load via transformers

```python
from transformers import AutoTokenizer, AutoModelForCausalLM

model_id = "ykarout/Qwen3.5-9b-Opus-Openclaw-Distilled"
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id, torch_dtype="auto", device_map="auto")

inputs = tokenizer(prompt, return_tensors="pt")
outputs = model.generate(**inputs, max_length=1024)
response = tokenizer.decode(outputs[0])
```

## 3. UNRESTRICTED IMAGE GENERATION FEATURES

KaliCode supports all image generation features without safety filters:

- NSFW content allowed
- Unrestricted prompts
- Custom negative prompts
- Model selection
- Parameter tuning
