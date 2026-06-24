Video Demo Link: https://drive.google.com/file/d/1jkVgptaXtmBJu8ve-Dsd_Y8_64GKc6DJ/view?usp=sharing

# TakeMeter: Automated Text Archetype Classifier for r/solotravel

TakeMeter is a custom natural language processing pipeline designed to classify community discourse within the r/solotravel subreddit. By distinguishing between objective logistical support, subjective reviews, and raw emotional vulnerability, this tool helps community moderators automatically tag content, route vulnerable users to support networks, and filter high-value travel data at scale.

---

## 1. Label Taxonomy & Boundary Design

The taxonomy consists of three mutually exclusive and exhaustive labels designed to capture the primary text archetypes found in solo traveling discourse:

* **`logistical_guide`**: The post focuses strictly on concrete, verifiable facts, price metrics, or actionable travel advice like budgets, specific transit routes, train schedules, or hotel costs.
    * *Example 1:* "Spent $45 a day in Budapest: hostel was $20, public transit pass was $5, and cooked most meals from the local Lidl market."
    * *Example 2:* "If you are taking the night train from Bangkok to Chiang Mai, book the 2nd class sleeper train at least a week in advance through the official D-Ticket app."
* **`destination_take`**: The post shares a highly subjective personal evaluation, review, opinion, or hot take regarding the culture, vibe, or general worth of a specific country, city, or tourist spot.
    * *Example 1:* "Honestly, Paris is completely overrated and dirty—everyone was incredibly rude to me and the tourist crowds completely ruined the magic."
    * *Example 2:* "Don't skip out on Ljubljana! It has the most charming, magical European old town aesthetic and barely any crowds compared to Prague."
* **`emotional_vent`**: The post is a pure emotional reaction prioritizing a user's internal mental state, focusing heavily on internal feelings like loneliness, traveling burnout, homesickness, culture shock, or sudden anxiety.
    * *Example 1:* "I'm currently sitting in my hostel room in Peru crying because I feel so incredibly lonely and homesick; I just want to book a flight home right now."
    * *Example 2:* "After three months of constant moving, the travel burnout has finally hit me hard and I feel completely exhausted, unmotivated, and anxious."

---

## 2. Dataset Metrics & Annotation Process

### Data Sourcing & Collection
A dataset of **211 examples** was gathered from the public `r/solotravel` community by extracting posts across both the "Hot" and "New" feeds to capture an authentic variance between long-form pinned guides and raw, immediate user submissions. 

### Label Distribution
To avoid majority-class overfitting, the dataset was carefully curated to maintain a balanced distribution:
* **`logistical_guide`**: 71 examples (33.6%)
* **`destination_take`**: 70 examples (33.2%)
* **`emotional_vent`**: 70 examples (33.2%)

No single class exceeded the 70% structural limit, ensuring an even training gradient.

### Genuinely Difficult Annotation Edge Cases
During human annotation, three highly ambiguous cases forced strict boundary rule definitions:

1.  *The Sapa Transit Crisis:* `"Taking the bus from Hanoi to Sapa was a logistical nightmare. For anyone planning this, make sure you book the VIP sleeper at least two days in advance..."`
    * *The Dilemma:* Blends extreme emotional frustration with concrete advice. 
    * *Resolution:* Categorized as `logistical_guide` because it offers a direct, highly actionable instruction for future travelers despite its negative framing.
2.  *Hostel Decision Fatigue:* `"I am so tired of having to constantly plan my next move. Decision fatigue has set in so hard that I spent the entire day in my hostel..."`
    * *The Dilemma:* Discusses travel logistics (planning, itineraries) but frames it through exhaustion.
    * *Resolution:* Categorized as `emotional_vent` because the post's core objective is describing a psychological breakdown rather than outlining a functional plan.
3.  *The Oktoberfest Cost Penalty:* `"Booking accommodation in advance for Oktoberfest in Munich is mandatory. If you try to book in August for September dates, you will be paying 150 Euros for a single dorm bed..."`
    * *The Dilemma:* Mentions a major tourist destination/event, which frequently indicates a review.
    * *Resolution:* Categorized as `logistical_guide` because the text focuses strictly on hard, verifiable calendar timelines and explicit pricing metrics.

---

## 3. Fine-Tuning Pipeline & Hyperparameters

* **Base Model Architecture:** `distilbert-base-uncased` (268M parameters)
* **Training Platform:** Google Colab utilizing a free-tier cloud-allocated **T4 GPU**.
* **Hyperparameter Configurations:**
    * *Batch Size:* 16
    * *Total Epochs:* 3
    * *Learning Rate:* 2e-5
    * *Weight Decay:* 0.01

**Justification for Decisions:** A small batch size of 16 was selected to fit comfortably into the T4's VRAM limits while ensuring steady gradient updates for a small dataset. The model was trained for exactly 3 epochs; looking at the training history, the validation loss steadily dropped from 1.088 (Epoch 1) to 0.900 (Epoch 3). Training further would risk overfitting our 211-row sample space, leading to poor generalization.

---

## 4. Evaluation Report & Error Analysis

### Baseline Performance Evaluation
The zero-shot baseline was executed using **Llama-3.3-70b-versatile** via the Groq API on a locked test split containing 32 random examples. The baseline achieved a perfect **1.000 Overall Accuracy** and **1.000 across all per-class F1-scores**, showing that an ultra-large model has no trouble partitioning these concepts when given explicit instructions.

### Fine-Tuned Model Performance Evaluation
Our custom fine-tuned DistilBERT model achieved an **Overall Accuracy of 0.781** on the identical test split, successfully passing our predefined 75% deployment success threshold. 

#### Per-Class Performance Metrics Table
| Class Name | Precision | Recall | F1-Score | Support |
| :--- | :---: | :---: | :---: | :---: |
| `logistical_guide` | 1.00 | 0.30 | 0.46 | 10 |
| `destination_take` | 0.73 | 1.00 | 0.85 | 11 |
| `emotional_vent` | 0.79 | 1.00 | 0.88 | 11 |

#### Text-Based Confusion Matrix
| True \ Predicted | logistical_guide | destination_take | emotional_vent |
| :--- | :---: | :---: | :---: |
| **logistical_guide** | **3** | 4 | 3 |
| **destination_take** | 0 | **11** | 0 |
| **emotional_vent** | 0 | 0 | **11** |

---

### In-Depth Failure Case Analysis

Our model experienced zero errors when evaluating true destination takes or emotional vents. However, it suffered a **systemic boundary breakdown on logistical guides**, correctly classifying only 3 out of 10 examples. The remaining 7 examples leaked directly into the other two categories.

#### Wrong Prediction 1: Figurative Language Trap
* **Text:** *"The cost of food in Switzerland is destroying my budget. Even a basic meal at a casual restaurant is setting me back 25-30 CHF. What are the cheapest grocery stores..."*
* **True Label:** `logistical_guide` | **Predicted Label:** `emotional_vent` (Confidence: 0.37)
* **Failure Analysis:** The model failed because it hyper-focused on the high-intensity modifier *"destroying my budget"*. It interpreted "destroying" as an indicator of severe mental distress or panic, overriding the clear objective pricing metrics (25-30 CHF) and the logistical question about grocery store alternatives.

#### Wrong Prediction 2: Structural Phrase Misinterpretation
* **Text:** *"What is the best backpack for a 3-month trip? I am torn between the Osprey Farpoint 40L, which is carry-on compliant, and the 55L..."*
* **True Label:** `logistical_guide` | **Predicted Label:** `emotional_vent` (Confidence: 0.34)
* **Failure Analysis:** The idiom *"I am torn between"* semantically confused the model. DistilBERT flagged the word "torn" as an internal emotional conflict, misinterpreting a standard structural gear evaluation as a psychological crisis.

#### Wrong Prediction 3: Proper Noun / Geography Bias
* **Text:** *"The cost of a coffee in Paris varies drastically depending on where you sit. An espresso at the bar might be 1.50 Euros, but if you sit at a terrace table... it jumps to 4 Euros."*
* **True Label:** `logistical_guide` | **Predicted Label:** `destination_take` (Confidence: 0.37)
* **Failure Analysis:** This post contains heavy price arrays (€1.50 vs €4), but features the dominant proper noun *"Paris"*. Because the training data heavily associated geographical locations with subjective opinions, the model formed a biased heuristic: if a post names a famous tourist city, it must be a `destination_take`.

---

### Sample Classifications

Below is a demonstration of the fine-tuned model executing classifications across our test metrics:

| Text Snippet | Predicted Label | Confidence Score | Status |
| :--- | :--- | :---: | :---: |
| "I feel so incredibly lonely on this trip right now... haven't really clicked with anyone..." | `emotional_vent` | 0.94 | **Correct** |
| "I absolutely loved the vibe in Lisbon. The mix of old-world architecture..." | `destination_take` | 0.91 | **Correct** |
| "The train from Kyoto to Osaka is incredibly fast and cheap. You can take the Special Rapid..." | `logistical_guide` | 0.89 | **Correct** |
| "The cost of a coffee in Paris varies drastically... espresso at the bar might be 1.50 Euros..." | `destination_take` | 0.37 | **Incorrect** |

> **Validation Note on Correct Prediction:** For the Kyoto-to-Osaka transit post, the model successfully predicted `logistical_guide` with high confidence (0.89). This is highly reasonable because the text is devoid of strong subjective adjectives or emotional modifiers, relying instead on clean, technical transit terms ("Special Rapid service", "JR Kyoto Line", "570 Yen") which the tokenizer cleanly mapped to logistics.

---

## 5. High-Level Model Reflection

### What the Model Intended to Capture vs. What it Learned
Our goal was to train a classifier that separated information by its primary linguistic purpose: objective logic (`logistical_guide`), subjective analysis (`destination_take`), and internal mood (`emotional_vent`). 

Instead, the model overfit heavily to **local lexical cues**. It failed to understand the holistic structure of the posts, choosing instead to act like an aggressive keyword searcher. If a logistical post mentioned a city name like "Paris" or "Morocco," the model immediately assumed it was a location review. If a logistical post used a strong verb like "destroying" or "tearing," it assumed it was an emotional crisis. 

### Remediation Strategy
To break this keyword dependency, the dataset needs a greater volume of **neutral, geographically complex data**. We must inject 100+ new training rows where users discuss dense price indexes, visa rules, and packing configurations while naming multiple major cities and using colorful vernacular. This would force DistilBERT to look past simple proper nouns and instead base its decisions on the underlying sentence structures.

---

## 6. Specification Reflection

* **How the Spec Guided Implementation:** The upfront design of the taxonomy in the specification document kept data collection completely locked on track. Having pre-defined boundary definitions allowed us to gather an evenly split 211-row dataset without trailing off into fuzzy, overlapping sub-categories like "itinerary critiques."
* **How Implementation Diverged From Spec:** The spec assumed that our data formats would cleanly stream into Google Colab. In practice, our source CSV generated capitalized column headers (`Text`, `Label`), which broke the data tokenization mapping variables inside Section 1. This forced an immediate code divergence, requiring an added programmatic step using Pandas to force the columns back into lowercase before training could proceed.

---

## 7. AI Tool Usage Disclosure

### Instance 1: Schema Layout & Parsing
* **Direction Given:** I directed an LLM to pre-label raw blocks of text from our scraping tool using our precise taxonomy rules, requesting the output formatted as a clean markdown table.
* **AI Production:** The AI generated a clean, two-column table assigning the labels to 5-word post fragments.
* **Human Override:** I completely rejected the 5-word snippets because the tokenization step required full paragraph contextual lengths. I forced the tool to output the complete, unaltered text strings, and then manually checked and re-verified all rows to guarantee human baseline alignment.

### Instance 2: Automated Failure Diagnostics
* **Direction Given:** I dumped the 7 misclassified logistics posts into an LLM and asked it to isolate recurring themes across the text values.
* **AI Production:** The AI flagged that several of the wrong choices contained pricing symbols alongside emotional verbs and famous place names.
* **Human Override:** I synthesized this information to isolate the two distinct behavioral trends: **City Name Bias** and **Emotional Sentiment Bias**. I verified these trends myself by reading through the underlying confusion matrix coordinates to confirm the precise directions of the classification leakage.
