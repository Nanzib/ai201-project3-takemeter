# TakeMeter Planning Specs

## 1. Community Context & Selection Analysis
* **Chosen Community:** r/solotravel (Reddit)
* **Why it was chosen:** This community is uniquely text-heavy, strictly moderated against low-effort images, and centers around deep personal text reflections. 
* **Suitability for Classification:** The discourse here is highly varied and emotionally complex. Unlike generic travel groups, a solo traveler's output naturally fractures into distinct archetypes based on their immediate psychological or analytical state. A user might post a meticulously structured, budget-capped transit schedule, change tabs to post an aggressive review arguing that a trendy city is a tourist trap, or write a deeply vulnerable post about experiencing severe panic and isolation in a hostel dorm. This sharp variance between pragmatic logic, opinionated critiques, and raw internal feelings makes it an ideal environment for testing a custom text classification model.

## 2. Label Taxonomy Design
*All labels are strictly designed to be mutually exclusive and exhaustive across the community's primary text formats.*

* **Label 0: logistical_guide**
  * *Definition:* The post focuses strictly on concrete, verifiable facts, price metrics, or actionable travel advice like currency limits, specific transit routes, or hostel bookings.
  * *Example 1:* "Spent $45 a day in Budapest: hostel was $20, public transit pass was $5, and cooked most meals from the local Lidl market."
  * *Example 2:* "If you are taking the night train from Bangkok to Chiang Mai, book the 2nd class sleeper train at least a week in advance through the official D-Ticket app."

* **Label 1: destination_take**
  * *Definition:* The post shares a highly subjective personal evaluation, review, or hot take regarding the culture, vibe, or general worth of a specific country or city.
  * *Example 1:* "Honestly, Paris is completely overrated and dirty—everyone was incredibly rude to me and the tourist crowds completely ruined the magic."
  * *Example 2:* "Don't skip out on Ljubljana! It has the most charming, magical European old town aesthetic and barely any crowds compared to Prague."

* **Label 2: emotional_vent**
  * *Definition:* The post is a pure emotional reaction prioritizing a user's internal mental state, focusing heavily on feelings like homesickness, traveling burnout, isolation, or sudden anxiety.
  * *Example 1:* "I'm currently sitting in my hostel room in Peru crying because I feel so incredibly lonely and homesick; I just want to book a flight home right now."
  * *Example 2:* "After three months of constant moving, the travel burnout has finally hit me hard and I feel completely exhausted, unmotivated, and anxious."

## 3. Hard Edge Cases & Boundary Rules
* **The Boundary Ambiguity:** The most frequent edge case occurs when a user links a terrible personal emotional event directly to a sweeping city review (e.g., *"I absolutely hated Rome because I got pickpocketed on my first day and spent the entire week feeling stressed, isolated, and miserable."*). It overlaps `destination_take` (declaring hatred for Rome) and `emotional_vent` (focusing on stress and isolation).
* **Defensive Boundary Rule:** If an evaluation of a location is entirely dependent on an isolated personal mishap or an internal mood state (like crime victimization, illness, or homesickness), it will be categorized strictly as `emotional_vent`. To qualify as a `destination_take`, the post must critique systemic elements of the destination itself (like city infrastructure, actual cost-of-living metrics, or crowds) rather than personal distress. The Rome example will therefore be annotated as `emotional_vent`.

## 4. Data Collection Plan
* **Source & Volume:** I will collect at least 210 public text rows directly from r/solotravel by pulling from both the "Hot" and "New" feeds to capture a healthy mix of pinned deep-dives and raw daily submissions.
* **Target Distribution:** I am aiming for an even split of roughly 70 examples per label to prevent class imbalance issues during training.
* **Underrepresentation Fallback:** If certain labels (like `emotional_vent`) are underrepresented after collecting 200 random rows, I will target specific keyword searches within the subreddit search bar (e.g., "lonely", "burnout", "scam" for emotional vents; "itinerary", "costs", "EUR" for logistical guides) to intentionally boost minority counts while maintaining authentic text formats.

## 5. Evaluation Metrics
* **Metrics Selected:** Overall Accuracy, Per-Class Precision, Per-Class Recall, and Macro F1-Score.
* **Justification:** Relying entirely on accuracy is dangerous for subjective text classification. For example, if our dataset naturally skews away from emotional crises, a model could score high accuracy by guessing `destination_take` every time. We need **Recall** on `emotional_vent` to ensure vulnerable posts are never missed, and we need **Precision** on `logistical_guide` to guarantee that a user looking for concrete transit data isn't served a subjective rant. The **Macro F1-Score** will serve as our primary structural health check to prove the model is performing evenly across all three distinct boundaries.

## 6. Definition of Success
* **Useful Threshold:** For this classifier to be genuinely helpful as an automated moderation or automated content-tagging tool in a real travel platform, it must hit an **Overall Accuracy of at least 75%** and a **Macro F1-Score of at least 72%**. 
* **Deployment Standard:** Traveling discourse is highly nuanced, so a 100% score is unrealistic. However, an accuracy of 75% ensures that three-quarters of the feed is categorized correctly without manual intervention, which is highly efficient for managing high-volume forum feeds at scale.

## 7. AI Tool Plan

* **Label Stress-Testing:** Before I begin manual annotation, I will prompt an LLM with my exact three label definitions and my edge-case boundary rule. I will direct it to intentionally generate 10 highly ambiguous, tricky travel posts that sit directly on the line between my classes. I will review its generation—if I cannot easily categorize those AI samples using my current rules, I will sharpen my boundary definitions before touching my real 200-row dataset.
* **Annotation Assistance:** To preserve the absolute integrity of my training data and eliminate model-inception bias, I will **not** use an LLM to pre-label my dataset. I will scrape the raw text and hand-annotate all 200+ examples myself to guarantee the labels perfectly align with human community interpretation.
* **Failure Analysis:** Once training finishes, I will download the list of misclassified text examples from Google Colab and feed them back into an LLM. I will ask the model to look for hidden linguistic patterns among the errors—such as whether our custom DistilBERT model consistently fails on sarcastic phrases, short sentence structures, or complex punctuation—and then manually verify those patterns against the true confusion matrix.

### Edge Cases Encountered

* **Difficult Post 1:** "Taking the bus from Hanoi to Sapa was a logistical nightmare. For anyone planning this, make sure you book the VIP sleeper at least two days in advance, otherwise you'll be stuck on the plastic floor mats."
  * *The Dilemma:* The post opens with strong emotional frustration ("logistical nightmare"), which strongly mimics an `emotional_vent`.
  * *Final Decision & Rationale:* Labeled as `logistical_guide`. Despite the emotional framing, the core intent of the post provides highly actionable, specific transit instructions (booking the VIP sleeper 2 days ahead) to future travelers.

* **Difficult Post 2:** "I am so tired of having to constantly plan my next move. Decision fatigue has set in so hard that I spent the entire day in my hostel simply because I couldn't choose a restaurant for lunch."
  * *The Dilemma:* This post actively discusses standard travel planning mechanics (choosing restaurants, moving locations), which borders on a `logistical_guide` or a general `destination_take`.
  * *Final Decision & Rationale:* Labeled as `emotional_vent`. The true focus of the text is not an evaluation of the restaurants or transit steps, but a deep dive into the user's internal psychological exhaustion and decision fatigue.

* **Difficult Post 3:** "Booking accommodation in advance for Oktoberfest in Munich is mandatory. If you try to book in August for September dates, you will be paying 150 Euros for a single dorm bed, if you can find one at all."
  * *The Dilemma:* This references a major travel destination event (Oktoberfest in Munich), which could easily make it a `destination_take`.
  * *Final Decision & Rationale:* Labeled as `logistical_guide`. The post relies entirely on concrete, verifiable timelines (August vs. September) and explicit cost metrics (€150 for a dorm bed) rather than subjective vibes.