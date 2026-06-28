![preview](https://raw.githubusercontent.com/AkshayPatil9370/picsart-genai-composer-sdk/main/preview.svg)

# VisualSynth Engine – Generative Media Orchestration Toolkit

**VisualSynth Engine** is a Python-native framework for orchestrating programmable image pipelines and generative AI workflows. Inspired by the creative API ecosystems that power modern media editing, this SDK abstracts away the complexity of chaining image transformation, enhancement, and generation calls into declarative, reusable sequences. Whether you are building a real-time background removal microservice, a multi-stage upscaling and enhancement queue, or a Text-to-Image campaign generator with style transfer layers, VisualSynth Engine provides the connective tissue between raw API endpoints and your application logic.

Built with the philosophy that creative automation should be both expressive and predictable, this toolkit offers a composable architecture where each image operation is a self-contained “node” that can be linked, tested, and reused across projects. The engine handles rate limiting, retry logic, batch parallelization, and output formatting so you can focus on the creative flow rather than boilerplate networking.

## Overview

Modern visual media workflows rarely consist of a single API call. They involve sequencing—remove a background, then apply an artistic effect, then upscale, then generate a caption. VisualSynth Engine treats each step as a discrete, chainable transformation within a “media pipeline.” Pipelines can be defined declaratively using YAML-like dictionaries or constructed programmatically via the fluent builder API.

The SDK is organized into three core layers:

- **Image Transformation Layer** – wrappers for programmable image APIs: background removal, upscaling, enhancement, color grading, effects application, and format conversion. Each operation supports configurable fidelity parameters (e.g., “preserve shadows” for background removal, “denoise strength” for upscale).
- **GenAI Generation Layer** – interfaces for Text-to-Image, Image-to-Image, inpainting, outpainting, and style transfer. Supports prompt engineering helpers, negative prompt management, and seed-controlled outputs for reproducibility.
- **Pipeline Orchestrator Layer** – the heart of the engine. Define a directed graph of operations, attach conditionals (e.g., “only upscale if image is below 2MP”), and execute with parallel branch support. The orchestrator outputs a structured provenance record so you can audit every transformation applied.

[![Download](https://raw.githubusercontent.com/AkshayPatil9370/picsart-genai-composer-sdk/main/button.svg)](https://akshaypatil9370.github.io/picsart-genai-composer-sdk/)

## Key Features

### 🧩 Declarative Pipeline Builder
Construct complex image workflows using a simple JSON or Python dictionary schema. Each node specifies an operation, its parameters, and its upstream dependencies. The engine validates the graph for cycles, missing inputs, and type mismatches before execution.

### ⚡ Parallel Branch Execution
When a pipeline contains independent branches (e.g., generating both a thumbnail and a full-resolution version of the same image), VisualSynth Engine dispatches them concurrently using an internal thread-based executor with configurable concurrency limits.

### 🔁 Automatic Retry with Exponential Backoff
Network hiccups and rate limits are inevitable. The SDK wraps every API call in a retry mechanism that respects `Retry-After` headers, implements jitter, and logs each attempt for debugging.

### 📦 Output Standardization
Every operation returns a unified `MediaAsset` object containing the binary data, MIME type, metadata (dimensions, color profile, generation parameters), and a provenance chain. This means downstream consumers always know the history of an asset.

### 🌐 Multilingual Prompt Support
A built-in prompt helper normalizes text across languages, automatically appending style cues and formatting for non-English prompts to maintain generation quality. Works with all Text-to-Image operations.

### 🛡️ Granular Error Handling
Errors are never opaque. The engine raises typed exceptions: `RateLimitError`, `InvalidParameterError`, `AuthenticationError`, `PipelineCycleError`. Each includes a machine-readable code and a human-readable explanation.

### 📊 Provenance Logging
Every pipeline execution generates a cryptographic-style provenance log (a JSON array of operation hashes). This enables full auditability—useful for compliance, debugging, and content authenticity verification.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Pipeline Definition                    │
│  { nodes: [{ op: "remove_bg", params: {…} }, …] }       │
└───────────────────────┬─────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────┐
│              Pipeline Orchestrator                       │
│  ├─ Validates DAG                                       │
│  ├─ Resolves dependencies                               │
│  ├─ Parallelizes independent branches                   │
│  └─ Manages state across nodes                          │
└───────────────────────┬─────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────┐
│            Transformation Node Executor                  │
│  ├─ Image Ops: remove_bg, upscale, enhance, effects     │
│  ├─ GenAI Ops: text2image, img2img, inpaint, style      │
│  ├─ Format Ops: resize, crop, convert, compress         │
│  └─ Each returns MediaAsset with provenance             │
└───────────────────────┬─────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────┐
│                 Unified Output Layer                     │
│  ├─ MediaAsset (bytes, mime, metadata, provenance)       │
│  ├─ Local file save                                     │
│  ├─ In-memory buffer for streaming                      │
│  └─ Webhook callback support                             │
└─────────────────────────────────────────────────────────┘
```

## Getting Started

VisualSynth Engine is designed to be integrated into any Python-based application—web backends, CLI tools, data pipelines, or interactive notebooks. The core dependency is minimal: the engine itself plus a lightweight HTTP client.

### Prerequisites

- Python 3.10 or later
- A valid API credential for the underlying creative API service (configured via environment variable or direct injection)

### Basic Usage

Define a pipeline that removes the background from an image and then upscales the result by 2x:

```python
from visualsynth import PipelineBuilder, MediaAsset

# Start with a source image
source = MediaAsset.from_file("portrait.jpg")

# Declare the pipeline
pipeline = (
    PipelineBuilder()
    .add_node("remove_bg", operation="remove_background", preserve_shadows=True)
    .add_node("upscale", operation="upscale", scale_factor=2, denoise=True)
    .link("remove_bg", "upscale")  # connect nodes
    .build()
)

# Execute synchronously
result = pipeline.run(source)

# The output is a MediaAsset with provenance
result.save("output.png")
print(result.metadata)  # dimensions, operations applied, timestamps
```

For parallel branches—generate a thumbnail while also enhancing the full image:

```python
pipeline = (
    PipelineBuilder()
    .add_node("enhance", operation="auto_enhance")
    .add_node("thumbnail", operation="resize", width=150, height=150)
    # Both nodes depend on the same source, so they run in parallel
    .build()
)

results = pipeline.run(source)
thumbnail = results["thumbnail"]
enhanced = results["enhance"]
```

## Pipeline Schema Reference

Pipelines can also be defined as a serializable dictionary, useful for storing configurations or sending workflows to a server.

```json
{
  "nodes": [
    {
      "id": "step_1",
      "operation": "remove_background",
      "params": {
        "preserve_shadows": true,
        "output_format": "png"
      }
    },
    {
      "id": "step_2",
      "operation": "text2image",
      "params": {
        "prompt": "A serene mountain landscape at sunset, digital art style",
        "negative_prompt": "blurry, low quality, distortion",
        "width": 1024,
        "height": 1024,
        "seed": 42
      }
    },
    {
      "id": "step_3",
      "operation": "composite",
      "params": {
        "blend_mode": "overlay",
        "opacity": 0.8
      }
    }
  ],
  "links": [
    {"from": "step_1", "to": "step_3"},
    {"from": "step_2", "to": "step_3"}
  ],
  "output": "composite"
}
```

The orchestrator validates this schema and resolves execution order.

## Error Handling

All errors are typed and carry an error code for programmatic handling:

```python
from visualsynth.exceptions import RateLimitError, PipelineCycleError

try:
    result = pipeline.run(source)
except RateLimitError as e:
    # Wait and retry, or switch to backup credential
    print(f"Rate limited: retry after {e.retry_after}s")
except PipelineCycleError as e:
    # The pipeline graph has a cycle—fix definition
    print(f"Cycle detected between nodes: {e.cycle_nodes}")
```

## Provenance and Audit Logs

Each `MediaAsset` carries a `provenance` attribute—a list of `OperationRecord` objects:

```python
for record in result.provenance:
    print(record.operation)        # e.g., "remove_background"
    print(record.timestamp)        # ISO 8601
    print(record.parameters)       # dict of parameters used
    print(record.input_hash)       # SHA-256 of input asset
    print(record.output_hash)      # SHA-256 of output asset
```

This enables full traceability of every transformation, which is critical for content authentication and compliance with disclosure regulations.

## Multilingual Support

The `PromptHelper` class normalizes text prompts across different languages:

```python
from visualsynth.utils import PromptHelper

helper = PromptHelper()
prompt_en = "a futuristic city skyline at night"
prompt_ja = "未来都市の夜景"

# Both produce high-quality results with appropriate style cues
final_prompt = helper.normalize(prompt_ja, language="ja", style="photorealistic")
```

The helper automatically appends known quality-enhancing terms based on the target language and artistic style.

## Use Cases

- **E-commerce media processing**: Automatically remove backgrounds, create white-background product shots, then generate multiple aspect-ratio variants for different marketplaces.
- **Content moderation pipelines**: Run images through enhancement, then generate a descriptive caption for review, all in a single declarative workflow.
- **Generative advertising campaigns**: Combine Text-to-Image generation with style transfer and color grading to produce brand-consistent visuals at scale.
- **Real-time video thumbnail generation**: Extract frames, upscale, apply effects, and composite text overlays in a sequential pipeline triggered by video upload events.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for the full text.

**MIT License**  
Copyright © 2026  

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

## Disclaimer

VisualSynth Engine is a software development toolkit that provides abstractions over third-party creative APIs. The quality, availability, and legality of content generated or processed through this SDK depend entirely on the underlying services and the user’s compliance with their terms of service. The creators of VisualSynth Engine make no guarantees regarding the output of any transformation or generation operation, and assume no liability for misuse of the toolkit, including but not limited to generating deceptive, harmful, or unauthorized content. Users are solely responsible for ensuring that their use cases comply with applicable laws and platform policies. The SDK does not store, transmit, or retain any user data or generated media beyond the execution context of the application in which it is embedded. All operations are performed ephemerally within the user’s environment, and no telemetry or analytics data is collected by the SDK itself.

## Support & Community

For questions, feature requests, and bug reports, please open an issue in the repository. We aim to respond within 48 hours during business days.

- Documentation: [https://visualsynth-engine.readthedocs.io](https://visualsynth-engine.readthedocs.io)
- Discussion forum: [https://github.com/visualsynth-engine/discussions](https://github.com/visualsynth-engine/discussions)

## Contributing

Contributions are welcome. Please review the [CONTRIBUTING.md](CONTRIBUTING.md) file for guidelines on coding standards, testing, and pull request workflows. All contributors must adhere to the Code of Conduct.

[![Download](https://raw.githubusercontent.com/AkshayPatil9370/picsart-genai-composer-sdk/main/button.svg)](https://akshaypatil9370.github.io/picsart-genai-composer-sdk/)