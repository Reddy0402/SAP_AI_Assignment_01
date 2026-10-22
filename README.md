# SAP_AI_Assignment_01
Classfication of Images using two models on coco dataset 


# Error-analysis notes

**LFM2.5-VL-1.6B**

- **Image 93 (laptop desk scene):** listed 24 classes that are not present. It appears to dump much of the class list instead of describing the image.
- **Image 28 (dining table):** fell into a repetition loop and never closed the array, so all 6 correct objects were lost as invalid.
- **Image 0 (living room):** looped on potted plant / dining table / chair / fireplace, and "fireplace" is outside the 80 classes.

**InternVL3.5-1B**

- **Image 84 (street, motorcycle):** added bench, stop sign, cell phone, potted plant and backpack, and missed bicycle and chair.
- **Image 28 (dining table):** used out-of-vocabulary names (tissue, bread, wine) and omitted person, so the output was invalid.
- **Image 0 (living room):** repeated "chair" until the token limit, giving invalid JSON.

---

# Results table

| Metric                    | LFM2.5-VL-1.6B | InternVL3.5-1B |
|:--------------------------|:---------------|:---------------|
| Precision                 | 0.494          | 0.800          |
| Recall (all / large only) | 0.135 / 0.162  | 0.264 / 0.310  |
| F1                        | 0.213          | 0.397          |
| Hallucination rate        | 9.0%           | 13.0%          |
| Valid-JSON rate           | 19.0%          | 42.0%          |
| Latency (s/image)         | 2.11           | 5.36           |
| Peak GPU memory (GB)      | 3.09           | 4.34           |

# Write-up questions

**1. Does LFM2.5's extra 0.6B parameters buy enough F1 and lower hallucination to justify its latency and memory?**

No. LFM2.5 is faster and lighter, but its F1 is much lower (0.213 vs 0.397), and once invalid outputs are excluded it hallucinates more than InternVL (47.4% vs 31.0%).

**2. Which model hallucinates more, and on what kinds of images?**

Strictly InternVL (13% vs 9%), but LFM2.5 is worse on valid outputs, mostly misidentifying household items. InternVL struggles on 2-3 object images, adding benches, laptops and potted plants.

**3. How much does recall change when small objects are ignored? What does that tell you about each model?**

Only slightly for both (LFM2.5 0.135 → 0.162, InternVL 0.264 → 0.310), so the real problem is invalid JSON (e.g. repeated text with no closing), not missed small objects.

**4. Which model would you deploy for real-time, high-volume tagging, and why?**

InternVL3.5-1B, because LFM's raw speed advantage mostly disappears once validity is counted (about 0.09 vs 0.08 valid images/s), and InternVL has much higher precision and F1. Constrained JSON decoding, a repetition penalty and a lower max_new_tokens would help both models.

---

# Supporting evidence

## Valid-only breakdown

| Metric                         | LFM2.5-VL-1.6B                       | InternVL3.5-1B                       |
|:-------------------------------|:-------------------------------------|:-------------------------------------|
| Valid outputs                  | 19 / 100                             | 42 / 100                             |
| Invalid reasons                | bad_json: 30, out_of_vocab: 51       | bad_json: 22, out_of_vocab: 36       |
| Precision (valid only)         | 0.494                                | 0.800                                |
| Recall (valid only)            | 0.848                                | 0.844                                |
| Hallucination rate (valid only)| 47.4%                                | 31.0%                                |

Typical invalid outputs: arrays that never close (repetition loops cut off at the token limit) and names outside the 80 classes (e.g. sky, tree, building, window).

## Model comparison

| Metric        | LFM2.5-VL-1.6B | InternVL3.5-1B | Better         |
|:--------------|:---------------|:---------------|:---------------|
| precision     | 0.494          | 0.800          | InternVL3.5-1B |
| recall_all    | 0.135          | 0.264          | InternVL3.5-1B |
| recall_large  | 0.162          | 0.310          | InternVL3.5-1B |
| f1            | 0.213          | 0.397          | InternVL3.5-1B |
| valid_json    | 0.190          | 0.420          | InternVL3.5-1B |
| hallucination | 0.090          | 0.130          | LFM2.5-VL-1.6B |
| latency       | 2.109          | 5.364          | LFM2.5-VL-1.6B |
| peak_gb       | 3.092          | 4.343          | LFM2.5-VL-1.6B |

**Wins:** InternVL3.5-1B 5, LFM2.5-VL-1.6B 3.

**F1 difference (LFM2.5 − InternVL):** −0.184, 95% CI [−0.285, −0.083]. The CI excludes 0, so the F1 gap is unlikely to be noise.

**Charts:** the notebook's "Quality (higher is better)" plot shows InternVL ahead on every quality metric; the "Cost / errors (lower is better)" plot shows LFM2.5 lower on latency and peak memory.
