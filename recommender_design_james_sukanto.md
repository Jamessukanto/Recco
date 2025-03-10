# Task Interpretation 

We might hold interview rounds to gather insights into the experience of property buying.  

The following, I think, are reasonable initial assumptions:

- **Contradictory Preferences**, because:
	- Users may not be sure of their preferences or priorities.  
	- Unconscious biases about what they value vs. what they think they should value.  
	- Users construct preferences over time
		- They don't initialize with a set of stable preferences
	- Examples:  
		- Claims to be “open to all neighborhoods” but consistently favorites 	properties within a small preferred zone.  
		- Claims “budget is x” but revises it in the face of a new investment opportunity

- **Fears Around a High-Stakes Decision**, because:
	- Property buying is a major financial commitment.  
	- Property misrepresentation.
		- Ties into the idea that “users don’t always mean what they say”, just as a property is often not what it seems.  



**Reinterpreting the task**, we then have:

  > Design a dynamic property recommender that:
  > - Allows users to be human ❤️: 
  >   - Let users change their mind.  
  >   - Let users be unsure of their preferences.  
  > - Eases concerns around property buying and builds confidence. 

<br>
<br>

# Architecture

![Architecture diagram](https://github.com/Jamessukanto/Recco/blob/main/diagram.png)

| Term | Examples | 
|------------|--------|
| Static user preference features | Bedrooms, location, budget, school ratings |  
| User events / interactions | Clicks, views, dwell time, etc. |  

<br>
<br>

## 0. Data Flow  🚀

> "There are about 50K listings in Singapore" - Property Guru

### (0.1) Airflow ETL 

**Extract:**  
- Static user preference data from app
- Async jobs to retrieve property data with Scrapy + Playwright  

**Transform:**  
- Standardize & generate **intermediate property vectors** :  
  - Property text descriptions, reviews → BERT embeddings  
  - Property images → CLIP embeddings  
  - Static user preference features   
  - Property proximity to amenities → Proximity features:  
    - Pre-fetch amenities with Google Places API  
    - Pre-compute & store proximity features  
    - Find nearby amenities with PostGIS  
    - Use Google Distance API for walking / driving distance  
    - Queries on new properties (no pre-computation):  
      - Index the property with H3  
      - Fetch amenities in the same + adjacent hexes  
      - Approximate haversine distance  

**Load:**  
- Intermediate property embeddings → FAISS  
- Intermediate property features → Postgres  

### (0.2) Kafka

**Input:** Streamed user events / interactions; Streamed market data  

**Load:**  
- User events → Postgres for periodic model retraining  
- User events → Redis cache for real-time use  
- Market features → Postgres:
  - Compute rolling average and deviation per region 

<br>
<br>

## 1. Preference Contradiction Resolver 🚀

We want to allow users to change their mind or be unsure of their preferences – we  surface preference contradictions so users can resolve them.

### (1.1) Preference Contradiction Detector  

**Input:** User events / interactions; Static user preferences features

**Logic:**  
- Rule-based checks to identify simple contradictions  
  - *e.g.* budget preference vs. viewed budget  
- LLM to identify and phrase implied contradictions  
  - *e.g.* area preference vs. viewed areas → surface to the user
- Cache surfaced contradictions

**Output:** (Sample UI nudge)
  > “You marked ‘open to any areas,’ but 80% of your views are in X. Should we adjust?”  

**Trigger:**  
- Only if a new pattern emerges, check against cache
- Only after X number of user clicks (subtle UI nudges)
-  "No user feedback is user feedback! ;) "  

**Evaluation:**  
- Metrics:
  - Precision: Measuring FPs ensures we aren’t nudging users unnecessarily
  - Recall: Ensures we don’t miss significant contradictions


<br>
<br>


## 2. Property Recommender  🚀
We want our recommender to account for a lot of factors: some static, some moving parts – we make use of a multi-modal setup containing: 
- Market Risk Appraiser
- Market-aware Content Based Filtering (CBF), with FAISS
- Transformer4Rec (T2R), session-based
- Session-aware Collaborative Filtering (CF)
- XGBoost, as final score aggregator


<br>


### (2.1) Market Risk Appraiser  

We want to ease user concerns around property buying by assigning a risk score.

**Input:** Market features

**Logic:**  Compute how extreme the price is, an initial, naive market_risk_score:
  - `sigmoid((price - moving_avg_in_region) / moving_std_in_region)`  

**Output:**  Market risk score (0 - 1). Optional UI display:  
  > “20% below market price”  


<br>


### (2.2) Market-Aware Content-Based Filtering (with FAISS)  

We recommend properties by comparing user profile to property embeddings, identifying properties that share similar attributes.

**Input:**  
- User interactions
- Intermediate property embeddings  
- Intermediate property features  
- Market risk score from (2.1)  

**Logic:**  
- Encode intermediate property vectors and market risk score into property embedding
- Learn user embeddings from interactions
- Perform ANN search in FAISS to retrieve properties

**Output:**  
- Similarity scores of properties  

**Evaluation:** 
Run offline evaluation (Pre-XGBoost): Use past interaction data to see how well CBF ranks relevant items.
  - Metrics:
    - On ranking quality:
      - MRR: How early the most relevant item appears in the ranked list
      - NDCG: Reward good placements of highly relevant items.
      - Diversity Score: Based on similarity between embeddings
      - Coverage Score: Avoid recommending the same few
    - On user engagement:
      - CTR: How often users click on recommended properties
    - Latency


<br>


### (2.3) Transformer4Rec

We recommend based on user interactions sequence within a session. <em>Transformer4Rec has nice integration with Hugging Face transformers!</em>

> The conventional primary output of a Transformer4Rec is typically the probability distribution over properties. 
> 
> We **discard this primary head, however, in favor of user session features**, as inputs to its downstream module i.e. Collaborative Filtering.

Not advisable as stand-alone, because:
- Users make big decisions over multiple sessions (months even).
- A single session's data is not sufficiently rich
- Transformer4Rec optimizes for real-time engagement

**Input:**  User events / interactions (from Reddis)

**Logic:**  
  - With multiple prediction heads, we do:
	- Binary classification: 
		- Predict if user initiate contact with an agent
		- Predict if user will revisit a previously viewed property
	- Regression:
		- Predict session duration
		- ...

**Output:**  User session features

**Evaluation:** 
Run offline evaluation (Pre-XGBoost): Use past interaction data to see how well CBF ranks relevant items.
  - Metrics: MSE for regressors, Cross entropy for classifiers


<br>


### (2.4) Session-Aware Collaborative Filtering (User-based)
We recommend based on what similar properties are engaged with.

**Input:** 
- User interactions
- User session features from (2.3)

**Logic:**  
- Assign different weights to interactions to generate engagement scores
- Create a user-property interaction matrix (rows=users; columns=properties, value=engagement scores)
- Apply Matrix Factorization (ALS) to uncover latent factors:  

| Method | ALS | SVD |  
|------------|--------|--------|  
| Scalability | Incremental | `O(N^3)` |  
| Adaptable to preference changes** | Updated without retraining | Batch → stale recommendations |  
| Treats missing data as | Potential +ve signals | Unknown |  

**Output:**  
- User engagement scores

**Evaluation:** 
The metrics and offline evaluation of (2.2) apply here


<br>


### (2.5) XGBoost

We aggregate scores to optimize property ranking.

> We opt for LightGBM if XGBoost proves slow.

**Input:**  
- Content-based filtering score from **(2.2)**  
- Collaborative filtering score from **(2.4)**  
- Redundant initial features (not intermediate embeddings!) 
	- Static user preference features
	- User events / interactions	

**Logic:**  
- Ensemble decision trees regress user engagement scores 
- Apply diversity scoring:
	- Determinantal Point Processes (DPP) over Max Marginal Relevance (MMR), because:
		- DPP is more rigorous, probabilistic tool – gives precise control over diversity for large datasets
		- MRR is heuristic-based. It's cheap but lower diversity guarantee

**Output:**  
- Final rank scores → Ranked recommendations
- Feature Importance

**Evaluation:**
Online evaluation: Use real user engagement on served recommendations
- Metrics: MSE 


<br>
<br>

## 3. Explainability AI  🚀

### (3.1) SHAP  

We build user confidence by being transparent about why we recommend the properties. 

> If SHAP proves slow, we substitute with:
> - KernelSHAP (approximation), or
> - Built-in XGBoost Feature Importance
> 	- No personalised explanations – only which features are important 

**Input:**  
- Trained **XGBoost**  
- XGBoost’s inputs  

**Logic:**  
- Track outputs by **perturbing each feature**  
- Assign **+/-ve impact scores** to features  

**Output:**  Sample eventual UI message (Ensure message is not complex to user):
> We recommend this because: 
> ✅ Near transit
> ✅ Users similar to you also checked it
> ❌ Slightly over budget


**Trigger:**
- Trigger upon explicit user ask, or
- Trigger for only top 4 recommendations
    
<br>
<br>


# Challenges  🔍 🐒

### Data Staleness  
- Issue: New stores open, hospitals shut down, etc.  
- Fix: Check `last_updated`, flag for re-fetch  

### Embedding Updates
- Issue: Recomputing embeddings at once is costly  
- Fix: Incremental updates when X amenities are added  

### Cold Starts (Sparse Features)
- Issue:
  - No initial user preferences / property data  
  - No initial user interaction history  
- Fix:  
  - Synthetic user profiles based on demographics  
  - Popularity-based recommendations  
  - Impute missing property features  
    - Use average, KNN, or values from similar properties  
  - Use embeddings instead of numeric features





