## 3. Answers for Custom VLM Design for Industrial Quality Inspection

### (A) Model Selection: Qwen-VL or Custom Modular
I would choose **Qwen2.5VL** as the foundation.

* **Why:** Unlike LLaVA, which uses a simple linear projection, Qwen-VL supports higher resolution inputs ($600 \times 600$+) and utilizes **coordinate tokens** (e.g., `<box>`) natively in its vocabulary. 
* **Factors:** * **Inference Speed:** The architecture supports 4-bit quantization, essential for <2s targets.
    * **Licensing:** It offers a commercially viable license for industrial applications.
    * **Localization:** It is pre-trained on grounding tasks, making it less prone to spatial "guessing" than BLIP-2.
* **Architectural Modifications:** To handle microscopic PCB defects, I would implement a **Global-Local Crop** strategy. The model should process the full PCB at low resolution and high-resolution "crops" of potential defect areas simultaneously.

---

### (B) Design Strategy: PCB-Specific Requirements
General VLMs fail on PCBs because defects are tiny relative to the board size.

* **Vision Encoder:** Use **SigLIP-SO400M**. It is more robust than standard CLIP and handles high-resolution patches more efficiently.
* **Language Decoder:** Use **Phi-3-Mini (3.8B)**. In an offline environment, a smaller, "smarter" LLM is faster and easier to keep on-device than a 7B+ model.
* **Fusion Mechanism:** Replace the standard projection with a **C-Abstractor**. This uses learnable queries to compress visual features into a fixed number of visual tokens, focusing the LLM's attention on relevant spatial anomalies rather than empty green space.



---

### (C) Optimization for <2s Offline Deployment
To meet the strict latency requirements on edge hardware:

1.  **Quantization (AWQ/GPTQ):** Compress the model to **4-bit**. This reduces VRAM usage significantly and speeds up the decoding phase.
2.  **FlashAttention-2:** Use optimized kernels to accelerate the attention mechanism.
3.  **TensorRT-LLM:** Deploy the final model using NVIDIA’s TensorRT-LLM engine, which optimizes the execution graph for the specific GPU used in the inspection line.
4.  **Static KV Caching:** Since inspection responses are structured and predictable, pre-allocating KV cache memory prevents allocation overhead during the <2s window.

---

### (D) Hallucination Mitigation

* **Task-Specific Vocabulary:** Restrict the decoder's output space using **constrained decoding** so it can only output valid defect classes or "None."
* **Negative Data Training:** Include more than 10000 "Clean" PCB images in the training set where the correct response is always a standardized "No defects detected." 
* **Loss Function:** Implement a **Localization-Aware Loss**. Penalize the model heavily if it describes a defect but provides coordinates that do not overlap with any visual features.

---

### (E) Training Plan

1.  **Stage 1: QA Generation (Offline):** Use a "Teacher" model to turn bounding box coordinates into natural language QA pairs. 
2.  **Stage 2: Pre-alignment:** Train the vision-language connector to align PCB visual features with the text embeddings.
3.  **Stage 3: Visual Instruction Tuning:** Fine-tune the whole system on the 50k generated QA pairs using **LoRA** to maintain the base model's reasoning while learning PCB specifics.

---

### (F) Validation
| Metric | Target | Description |
| :--- | :--- | :--- |
| **mAP @ IoU 0.5** | $>0.92$ | Accuracy of the bounding boxes mentioned in text. |
| **Counting Accuracy** | $>98\%$ | Ability to correctly count multiple defects (e.g., "3 missing resistors"). |
| **Hallucination Rate** | $<1\%$ | Frequency of reporting defects on clean boards. |
| **Inference Latency** | $<1.8s$ | End-to-end time from image input to structured string output. |
