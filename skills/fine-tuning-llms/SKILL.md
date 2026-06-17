---
name: fine-tuning-llms
description: Expert guidance for LLM fine-tuning decisions, PEFT methods (LoRA/QLoRA), dataset preparation, training, evaluation, and deployment — invoke when the user asks about training, adapting, or customizing a language model.
---

# LLM Fine-Tuning Expert

## Decision Matrix: Prompting vs RAG vs Fine-Tuning

Before writing a single line of training code, answer these questions:

```
Is the task failing because the model lacks KNOWLEDGE?
    YES --> Use RAG (retrieval, not fine-tuning)
    NO  --> Is the task failing because of FORMAT, STYLE, or TONE?
                YES --> Fine-tuning is a strong candidate
                NO  --> Is prompt engineering fully exhausted?
                            NO  --> Start with prompting (cheapest, fastest)
                            YES --> Fine-tune
```

### Detailed Decision Table

| Criterion                           | Prompting | RAG  | Fine-Tuning |
|-------------------------------------|-----------|------|-------------|
| Knowledge is static / baked-in      | OK        | No   | Yes         |
| Knowledge is external / updatable   | No        | Yes  | No          |
| Knowledge corpus > context window   | No        | Yes  | No          |
| Style / tone / format consistency   | Partial   | No   | Yes         |
| Task requires labeled examples      | No        | No   | Yes         |
| Latency / cost requires small model | No        | No   | Yes         |
| General capability must be retained | Yes       | Yes  | Risk        |
| Time-to-first-result               | Hours     | Days | Weeks       |

**Anti-pattern**: using fine-tuning to inject factual knowledge.
The model will hallucinate with more confidence, not less.
Use RAG for facts. Use fine-tuning for behavior.

---

## PEFT: Parameter-Efficient Fine-Tuning

### Full Fine-Tuning vs PEFT

```
Full Fine-Tuning
================
All W updated   --> 7B model = ~28 GB of gradients alone
Catastrophic forgetting risk
Requires A100 80GB+ for any serious model
Learning rate must be tiny (1e-5 to 5e-5)

PEFT
====
Freeze W_base; add small trainable delta
90-99% fewer trainable parameters
Same or better quality than full FT for most tasks
Fits on a single consumer GPU with quantization
```

### LoRA (Low-Rank Adaptation)

For a weight matrix W (d x k), instead of updating W directly,
LoRA injects:

```
W' = W + (B * A)   where A is (r x k), B is (d x r), r << d
```

Only A and B are trained. W is frozen.

```
Typical LoRA settings
---------------------
r (rank):    8  -- minimal, fast, good baseline
             16 -- good default for most tasks
             32 -- more capacity, text-heavy tasks
             64 -- complex tasks, longer to train
             128 -- rarely needed; diminishing returns

lora_alpha:  equal to r (effective lr scale = 1.0)
             2x r (effective lr scale = 2.0, more aggressive)

lora_dropout: 0.0 for inference-focused tasks
              0.05-0.1 if overfitting observed

target_modules (order by impact):
  1. q_proj, v_proj           -- minimum viable
  2. + k_proj, o_proj         -- recommended default
  3. + gate_proj, up_proj, down_proj  -- FFN layers; max coverage
  4. embed_tokens, lm_head    -- rarely needed
```

### QLoRA (Quantized LoRA)

Quantize base model to 4-bit NF4, run LoRA on top.
Fits 70B model on a single A100 80GB.

```
Memory comparison (Llama-3 70B):
  Full FT:    ~560 GB VRAM  (fp32 weights + optimizer)
  LoRA bf16:  ~140 GB VRAM
  QLoRA 4bit: ~48 GB VRAM  <-- single A100 80GB fits this
```

Key QLoRA settings:
- `bnb_4bit_quant_type="nf4"` -- NormalFloat4; better than int4 for LLMs
- `bnb_4bit_compute_dtype=torch.bfloat16` -- upcast for compute
- `bnb_4bit_use_double_quant=True` -- quantize the quantization constants

### DoRA vs IA3

| Method | Params  | When to use                               |
|--------|---------|-------------------------------------------|
| LoRA   | Low     | Default choice; well-tested               |
| QLoRA  | Low     | Memory-constrained; 70B+ models           |
| DoRA   | Low     | When LoRA underfits; decomposes magnitude + direction |
| IA3    | Minimal | Style transfer; very few params; less flexible |

---

## Dataset Preparation

### Formats

**Instruction format** (Alpaca-style):
```json
{
  "instruction": "Summarize the following customer complaint in one sentence.",
  "input": "I ordered the blue widget on March 3rd and it arrived broken...",
  "output": "Customer received a broken blue widget and requests a replacement."
}
```

**Chat format** (preferred for instruct models):
```json
{
  "messages": [
    {"role": "system",  "content": "You are a customer support specialist..."},
    {"role": "user",    "content": "I ordered the blue widget..."},
    {"role": "assistant","content": "I'm sorry to hear that. Let me..."}
  ]
}
```

### Quality Checklist

```
[ ] Minimum 500 examples; 1000-5000 for best results
[ ] Quality >> quantity: 300 great examples beat 3000 mediocre ones
[ ] Each output is something you are proud to show a customer
[ ] Distribution covers all sub-tasks, not just the easy majority case
[ ] No near-duplicates (use MinHash or simhash to deduplicate)
[ ] No examples where the labeled output is wrong (filter aggressively)
[ ] No PII in training data unless explicitly required and consented
[ ] 90% train / 10% validation split, stratified if possible
[ ] Validation set has NO overlap with train (check by exact hash)
[ ] Edge cases and failure modes are explicitly represented
```

### Anti-Patterns in Dataset Prep

- **Length bias**: all outputs same length causes model to always generate that length
- **Copy-paste duplicates**: model memorizes, doesn't generalize
- **Majority class dominance**: if 80% of examples are "category A", model predicts A always
- **Inconsistent labeling**: same input, different output -- model learns noise
- **System prompt leakage in output**: output contains fragments of the instruction

---

## Training Code

### Python: Unsloth + TRL (recommended for speed)

```python
from unsloth import FastLanguageModel
from trl import SFTTrainer, SFTConfig
from datasets import load_dataset
import torch

# 1. Load model with 4-bit quantization
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/llama-3-8b-bnb-4bit",
    max_seq_length=2048,
    dtype=None,           # auto-detect
    load_in_4bit=True,
)

# 2. Attach LoRA adapter
model = FastLanguageModel.get_peft_model(
    model,
    r=16,
    target_modules=["q_proj", "v_proj", "k_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    lora_alpha=16,
    lora_dropout=0,       # 0 is optimized by Unsloth
    bias="none",
    use_gradient_checkpointing="unsloth",  # long context support
    random_state=42,
)

# 3. Load and format dataset
dataset = load_dataset("json", data_files="train.jsonl", split="train")

def format_chat(example):
    return tokenizer.apply_chat_template(
        example["messages"],
        tokenize=False,
        add_generation_prompt=False,
    )

dataset = dataset.map(lambda x: {"text": format_chat(x)})

# 4. Configure trainer
trainer = SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataset=dataset,
    args=SFTConfig(
        dataset_text_field="text",
        max_seq_length=2048,
        per_device_train_batch_size=2,
        gradient_accumulation_steps=8,   # effective batch = 16
        warmup_steps=50,
        num_train_epochs=2,
        learning_rate=2e-4,
        fp16=not torch.cuda.is_bf16_supported(),
        bf16=torch.cuda.is_bf16_supported(),
        logging_steps=10,
        save_strategy="epoch",
        output_dir="./outputs",
        lr_scheduler_type="cosine",
        seed=42,
    ),
)

# 5. Train
trainer.train()

# 6. Merge and save
model = FastLanguageModel.for_inference(model)
model.save_pretrained_merged("model-merged", tokenizer, save_method="merged_16bit")
# Or save as GGUF for llama.cpp:
# model.save_pretrained_gguf("model-gguf", tokenizer, quantization_method="q4_k_m")
```

### Python: Hugging Face PEFT + Transformers (more control)

```python
from transformers import (
    AutoModelForCausalLM, AutoTokenizer,
    TrainingArguments, Trainer, BitsAndBytesConfig
)
from peft import LoraConfig, get_peft_model, TaskType, prepare_model_for_kbit_training
import torch

# 1. BitsAndBytes config for QLoRA
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.1-8B-Instruct",
    quantization_config=bnb_config,
    device_map="auto",
    attn_implementation="flash_attention_2",
)
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.1-8B-Instruct")
tokenizer.pad_token = tokenizer.eos_token

# 2. Prepare for k-bit training (adds gradient cast hooks)
model = prepare_model_for_kbit_training(model)

# 3. LoRA config
lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "v_proj", "k_proj", "o_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type=TaskType.CAUSAL_LM,
)
model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Output: trainable params: 20,971,520 || all params: 8,051,232,768 || trainable%: 0.26

# 4. Training args
training_args = TrainingArguments(
    output_dir="./outputs",
    num_train_epochs=2,
    per_device_train_batch_size=2,
    gradient_accumulation_steps=8,
    learning_rate=2e-4,
    lr_scheduler_type="cosine",
    warmup_ratio=0.05,
    bf16=True,
    logging_steps=10,
    save_strategy="epoch",
    evaluation_strategy="epoch",
    load_best_model_at_end=True,
    report_to="wandb",
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
)
trainer.train()
```

### Java/Kotlin: Training via DJL (Deep Java Library)

```kotlin
import ai.djl.Model
import ai.djl.training.DefaultTrainingConfig
import ai.djl.training.loss.Loss
import ai.djl.training.optimizer.Optimizer
import ai.djl.training.tracker.Tracker

// DJL wraps PyTorch; Python is the primary fine-tuning surface.
// For Java/Kotlin production, preferred pattern is:
// 1. Fine-tune in Python (Unsloth/TRL)
// 2. Export to ONNX or GGUF
// 3. Serve via DJL or llama.cpp JNI bindings

class FineTunedModelLoader(private val modelPath: String) {

    fun loadMergedLoRA(): Model {
        // Load ONNX-exported merged model
        val criteria = Criteria.builder()
            .setTypes(Input::class.java, Output::class.java)
            .optModelPath(Paths.get(modelPath))
            .optEngine("OnnxRuntime")
            .build()
        return ModelZoo.loadModel(criteria)
    }
}

// Serving a GGUF model via llama.cpp bindings (Kotlin)
class LlamaCppInference(modelPath: String) {
    private val model = LlamaModel(
        LlamaModelParams()
            .nGpuLayers(35)    // offload layers to GPU
            .seed(42)
    )

    fun generate(prompt: String, maxTokens: Int = 512): String {
        val params = LlamaInferenceParams()
            .temperature(0.7f)
            .topP(0.9f)
            .maxNewTokens(maxTokens)
        return model.generate(prompt, params)
    }
}
```

---

## Hyperparameter Reference

```
Parameter              | Recommended Range        | Notes
-----------------------|--------------------------|----------------------------------
learning_rate          | 1e-4 to 3e-4 (LoRA)     | Higher than full FT; cosine decay
                       | 1e-5 to 5e-5 (full FT)  |
num_train_epochs       | 1 to 3                   | Watch val loss; stop early
per_device_batch_size  | 1 to 4                   | Limited by VRAM
gradient_accumulation  | 4 to 16                  | Effective batch = device * accum
warmup_ratio           | 0.03 to 0.10             | 3-10% of total steps
max_seq_length         | Match use case           | 2048 default; 4096+ for long docs
weight_decay           | 0.0 to 0.1               | Light regularization
max_grad_norm          | 1.0                      | Clip gradients; prevent explosion
lora_r                 | 8 to 64                  | 16 is a strong default
lora_alpha             | r to 2*r                 | alpha/r = scaling factor
```

**Early stopping**: stop when val loss stops decreasing for 2+ eval steps.
Do not continue training past the val loss minimum.

---

## Evaluation Framework

### Quantitative Metrics

```python
from evaluate import load

# For text generation tasks
rouge = load("rouge")
bleu  = load("bleu")
bertscore = load("bertscore")

results = rouge.compute(
    predictions=generated,
    references=references,
    use_aggregator=True,
)
# Key: rouge2 F1 for summarization; rougeL for seq-to-seq

# For classification fine-tuning
accuracy = load("accuracy")
f1 = load("f1")
```

### Qualitative Evaluation

```
Blind A/B Test Protocol
-----------------------
1. Sample 50-100 held-out examples not in train or val
2. Generate responses from: base model, fine-tuned model
3. Randomize ordering so evaluator doesn't know which is which
4. Score each response on: accuracy, format, tone (1-5 scale)
5. Report win rate: fine-tuned wins / total comparisons

Minimum bar: fine-tuned model should win >= 70% of comparisons
Red flag:    fine-tuned model wins < 55% -- training was ineffective
```

### Regression Testing

```python
# Always test general capability after fine-tuning
# Use lm-evaluation-harness (EleutherAI)

# Run before fine-tuning, record scores
# Run after fine-tuning, compare

benchmarks_to_run = [
    "mmlu",          # General knowledge
    "hellaswag",     # Common sense reasoning
    "arc_challenge", # Science QA
    "truthfulqa_mc", # Factuality
]

# Acceptable regression: < 2 points on any benchmark
# Red flag: > 5 points drop = catastrophic forgetting
```

### Overfitting Signals

```
Training loss falls  |  Val loss falls  --> Learning; continue
Training loss falls  |  Val loss flat   --> Marginal; consider stopping
Training loss falls  |  Val loss rises  --> OVERFITTING; stop immediately
Training loss flat   |  Val loss flat   --> Not learning; check data/LR
```

---

## Deployment

### Step 1: Merge LoRA Weights

```python
from peft import PeftModel
from transformers import AutoModelForCausalLM

# Load base + adapter, then merge
base = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3.1-8B-Instruct")
model = PeftModel.from_pretrained(base, "./lora-adapter")
model = model.merge_and_unload()  # LoRA weights folded into base
model.save_pretrained("./merged-model")
```

### Step 2: Quantize for Serving

```bash
# GGUF for llama.cpp (CPU or Apple Silicon)
python llama.cpp/convert.py ./merged-model --outfile model.gguf --outtype f16
./llama.cpp/quantize model.gguf model-q4_k_m.gguf Q4_K_M

# AWQ for fast GPU inference
pip install autoawq
python -c "
from awq import AutoAWQForCausalLM
model = AutoAWQForCausalLM.from_pretrained('./merged-model')
model.quantize(tokenizer, quant_config={'zero_point': True, 'q_group_size': 128, 'w_bit': 4})
model.save_quantized('./model-awq')
"
```

### Step 3: Serve with vLLM

```python
# vLLM: continuous batching + PagedAttention; highest throughput
from vllm import LLM, SamplingParams

llm = LLM(
    model="./merged-model",
    tensor_parallel_size=1,     # number of GPUs
    gpu_memory_utilization=0.90,
    max_model_len=2048,
)

params = SamplingParams(temperature=0.7, top_p=0.9, max_tokens=512)
outputs = llm.generate(["Summarize: ..."], params)

# Serving multiple LoRA adapters WITHOUT merging:
llm = LLM(
    model="meta-llama/Llama-3.1-8B-Instruct",
    enable_lora=True,
    max_lora_rank=64,
)
# Per-request adapter selection
from vllm.lora.request import LoRARequest
outputs = llm.generate(
    prompts,
    params,
    lora_request=LoRARequest("adapter-name", 1, "./lora-adapter"),
)
```

---

## Anti-Patterns Checklist

```
[ ] NOT using fine-tuning to inject new factual knowledge (use RAG)
[ ] NOT skipping a prompting baseline before spending on fine-tuning
[ ] NOT training on your full dataset without a held-out val split
[ ] NOT ignoring val loss (only watching train loss)
[ ] NOT training more epochs after val loss starts rising
[ ] NOT serving raw LoRA adapter without merging (or without vLLM adapter support)
[ ] NOT skipping regression tests on general benchmarks
[ ] NOT using fine-tuning to fix a prompting problem
[ ] NOT putting PII or secrets in training data
[ ] NOT choosing rank=256 because "more is better" -- diminishing returns
```

---

## Quick Reference: Hardware Requirements

```
Model Size | Method       | Min VRAM   | Recommended
-----------|--------------|------------|------------------
7-8B       | QLoRA 4bit   | 8 GB       | RTX 3090 (24 GB)
7-8B       | LoRA bf16    | 24 GB      | A10G (24 GB)
13B        | QLoRA 4bit   | 12 GB      | RTX 3090 (24 GB)
70B        | QLoRA 4bit   | 48 GB      | A100 80GB
70B        | LoRA bf16    | 160 GB     | 2x A100 80GB
```
