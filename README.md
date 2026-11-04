# two-stage-recsys

A two-stage product recommender using two-tower retrieval and LightGBM ranking, served with FastAPI and Docker.
Dataset: [Amazon Reviews 2023 — Video_Games](https://amazon-reviews-2023.github.io/)

> **Status:** work in progress. Currently exploring the data.

## Problem

Given a user's review history, recommend a ranked list of video games they are likely to enjoy.

Real users can't be asked what they would like, so the project is evaluated by hiding the future: games a user reviewed _after_ a cutoff date stand in for "games they would have liked", and the system is scored on whether it ranks those games near the top.

## Data

- **Source:** Amazon Reviews 2023 (McAuley Lab, UCSD), Video_Games category, 5-core benchmark files.
- **5-core:** every user and every game has at least 5 interactions. Per the dataset site: ~94.8K users, ~25.6K games, ~815K interactions.
- **Time-based split** (the dataset's absolute-timestamp split):
  - train: before ~Aug 2021 (`t_1 = 2021-08-11 00:56:54 UTC`)
  - validation: ~Aug 2021 to ~Jul 2022 (`t_2 = 2022-07-16 20:18:49 UTC`)
  - test: ~Jul 2022 to Sep 2023
- **Columns:** `user_id`, `parent_asin` (the game), `rating`, `timestamp` (milliseconds), `history` (the user's earlier games).
- **The data is not stored in this repo.** Download `Video_Games.train.csv`, `Video_Games.valid.csv` and `Video_Games.test.csv` from the `benchmark/5core/timestamp_w_his` folder of the [Hugging Face dataset](https://huggingface.co/datasets/McAuley-Lab/Amazon-Reviews-2023) into `data/raw/`.

## Approach

1. **Baselines.** Most-popular games and a simple "games reviewed together" method, to show the models add value.
2. **Stage 1: retrieval (two-tower + FAISS).** A user tower turns a user's history into a vector and an item tower turns each game into a vector, both trained on how all users behaved. Game vectors are computed once and stored in a FAISS index. At request time the user's vector pulls the closest few hundred candidate games out of ~25.6K.
3. **Stage 2: ranking (LightGBM).** A learning-to-rank model re-orders the candidates using richer features (for example popularity, average rating computed from training data only, price and category from item metadata, and the Stage 1 similarity score) to produce the final list.
4. **Serving.** A FastAPI service packaged in Docker: send a user's history, get back a ranked list of games.

## Evaluation

- Rank the **full catalog** (all ~25.6K games), no sampled negatives.
- Metrics: **Recall@20** and **NDCG@10** on the test period.
- Compare baselines vs. Stage 1 alone vs. Stage 1 + Stage 2.
- Also compare exact search against FAISS (recall lost vs. speed gained).

## Open questions (to answer during data exploration)

- **Ratings.** A 1-star review means the user was unhappy, so it should not count the same as a 5-star one. Leaning towards treating only higher ratings (e.g. 4 and above) as positives, for training and for evaluation, and using rating-based features in Stage 2. To confirm after looking at the rating distribution.
- **Users with no history.** Some test users have an empty `history` (their first interaction is in the test period). What does the system recommend for them?
- **Evaluation unit.** For a test row, `history` seems to include the user's earlier test-period interactions. Is each test row its own query, or is each user?
- **Already-seen games.** Should games already in a user's history be removed from the recommendations?
- **Unseen games.** How many test games never appear in training? This limits the best possible Recall@20.

## Decisions

_None yet. Each decision is recorded here, with the reason, as it is made._

## Progress

- [x] Dataset downloaded, repo and environment set up
- [ ] Data exploration
- [ ] Evaluation setup
- [ ] Baselines
- [ ] Stage 1: two-tower + FAISS
- [ ] Stage 2: LightGBM ranker
- [ ] FastAPI + Docker

## Dataset citation

Hou, Y., Li, J., He, Z., Yan, A., Chen, X., McAuley, J. (2024). _Bridging Language and Items for Retrieval and Recommendation._ arXiv:2403.03952.
