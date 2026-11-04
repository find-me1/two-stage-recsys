# two-stage-recsys

A two-stage product recommender using two-tower retrieval and LightGBM ranking, served with FastAPI and Docker.
Dataset: [Amazon Reviews 2023 — Video_Games](https://amazon-reviews-2023.github.io/)

> **Status:** work in progress. Data exploration done; next: item metadata check and evaluation setup.

## Problem

Given a user's review history, recommend a ranked list of video games they are likely to enjoy.

Real users can't be asked what they would like, so the project is evaluated by hiding the future: games a user reviewed _after_ a cutoff date stand in for "games they would have liked", and the system is scored on whether it ranks those games near the top.

## Data

- **Source:** Amazon Reviews 2023 (McAuley Lab, UCSD), Video_Games category, 5-core benchmark files.
- **5-core:** every user and every game has at least 5 interactions.
- **Time-based split** (the dataset's absolute-timestamp split):
  - train: before ~Aug 2021 (`t_1 = 2021-08-11 00:56:54 UTC`)
  - validation: ~Aug 2021 to ~Jul 2022 (`t_2 = 2022-07-16 20:18:49 UTC`)
  - test: ~Jul 2022 to Sep 2023
- **Columns:** `user_id`, `parent_asin` (the game), `rating`, `timestamp` (milliseconds), `history` (the user's earlier games).
- **The data is not stored in this repo.** Download `Video_Games.train.csv`, `Video_Games.valid.csv` and `Video_Games.test.csv` from the `benchmark/5core/timestamp_w_his` folder of the [Hugging Face dataset](https://huggingface.co/datasets/McAuley-Lab/Amazon-Reviews-2023) into `data/raw/`.

## Findings

From [`notebooks/01_explore_data.ipynb`](notebooks/01_explore_data.ipynb):

- Files match the dataset page: 736,827 / 34,510 / 43,249 rows (train / valid / test). The cutoffs split cleanly: train before 2021-08-11, valid until 2022-07-16, test until 2023-09-02.
- 94,762 users, 25,612 games, ~815K interactions. No duplicate user–game pairs; every user and game has at least 5 rows.
- A missing `history` marks a user's first review (12.4% of train rows, ~4% of valid and test).
- Users: median 6 rows, mean 8.6, max 473. Most users sit near the 5-review floor, with a few heavy users. In valid/test most users have 1–2 rows; the heaviest has 184.
- Games: median 12 reviews, mean 31.8, max 2,955. The top 1% of games (256) hold 18.6% of all rows.
- Ratings: 64% five-star, 17% four-star, 20% three-star or lower, similar in train and test.
- Cold-start users: 1,416 valid and 2,680 test users never appear in train. By rows, 41% of test rows (32% against train+valid) come from users with no earlier training data.
- Cold-start games: 27.9% of valid rows and 60.5% of test rows involve a game not in train. Even with valid added, 43.0% of test rows (1,542 games) do. An ID-only model can hit at most ~57% of test rows.

## Approach

1. **Baselines.** Most-popular games and a simple "games reviewed together" method, to show the models add value.
2. **Stage 1: retrieval (two-tower + FAISS).** A user tower turns a user's history into a vector and an item tower turns each game's ID and content (title, category, price) into a vector, both trained on how all users behaved. Game vectors are computed once and stored in a FAISS index. At request time the user's vector pulls the closest few hundred candidate games out of ~25.6K.
3. **Stage 2: ranking (LightGBM).** A learning-to-rank model re-orders the candidates using richer features (for example popularity, average rating computed from training data only, price and category from item metadata, and the Stage 1 similarity score) to produce the final list.
4. **Serving.** A FastAPI service packaged in Docker: send a user's history, get back a ranked list of games.

## Evaluation

- Rank the **full catalog** (all ~25.6K games), no sampled negatives.
- Metrics: **Recall@20** and **NDCG@10** on the test period.
- Compare baselines vs. Stage 1 alone vs. Stage 1 + Stage 2.
- Also compare exact search against FAISS (recall lost vs. speed gained).
- Games already in a user's history are removed before ranking.
- Report rows whose game was seen in training separately from rows with new games (43% of test rows involve a game unseen in train+valid).
- Score per row and per user (a few users have up to 184 test rows). Rows with no history (~4% of test) are reported separately.

## Decisions

- **Positives:** ratings of 4 and above count as "liked" (keeps ~80% of rows). Provisional; compare against using all interactions later.
- **Item tower uses content features** (title, category, price, store), so games with no training reviews can still be recommended.
- **User tower is built from `history`,** not a per-user ID embedding, because a third of test rows come from users with no earlier training data.
- **Games already in a user's history are excluded from recommendations.** No user reviews the same game twice, so recommending one only wastes a slot.

## Open questions

- **Metadata coverage.** Do the ~1.5K games unseen in train+valid have item metadata? The content-feature approach depends on it.
- **Users with no history.** ~4% of test rows are a user's first-ever review. What does the system recommend for them?
- **Evaluation unit.** Score per row, per user, or both?
- **Heavy users.** Cap or down-weight users with hundreds of reviews in training?
- **Metadata leakage.** Item rating statistics in the metadata include reviews up to Sep 2023. Compute them from the training period instead.

## Limitations

- 5-core keeps only active users and games with enough reviews, so results look better than a real cold-start setting.
- A review shows interest, not purchase or enjoyment. Offline evaluation can't see games a user would have liked but never reviewed.
- Reviewers are a self-selected group, so conclusions apply to people who review.

## Progress

- [x] Dataset downloaded, repo and environment set up
- [x] Data exploration
- [ ] Item metadata check
- [ ] Evaluation setup
- [ ] Baselines
- [ ] Stage 1: two-tower + FAISS
- [ ] Stage 2: LightGBM ranker
- [ ] FastAPI + Docker

## Dataset citation

Hou, Y., Li, J., He, Z., Yan, A., Chen, X., McAuley, J. (2024). _Bridging Language and Items for Retrieval and Recommendation._ arXiv:2403.03952.
