# two-stage-recsys

A two-stage product recommender using two-tower retrieval and LightGBM ranking, served with FastAPI and Docker.
Dataset: [Amazon Reviews 2023 — Video_Games](https://amazon-reviews-2023.github.io/)

> **Status:** work in progress. Data and item metadata exploration done; next: evaluation setup and baselines.

## Problem

Given a user's review history, recommend a ranked list of items from the Video Games catalog (games, accessories and gaming hardware) they are likely to want.

Real users can't be asked what they would like, so the project is evaluated by hiding the future: items a user reviewed _after_ a cutoff date stand in for "items they would have liked", and the system is scored on whether it ranks those items near the top.

## Data

- **Source:** Amazon Reviews 2023 (McAuley Lab, UCSD), Video_Games category, 5-core benchmark files.
- **5-core:** every user and every item has at least 5 interactions.
- **Time-based split** (the dataset's absolute-timestamp split):
  - train: before ~Aug 2021 (`t_1 = 2021-08-11 00:56:54 UTC`)
  - validation: ~Aug 2021 to ~Jul 2022 (`t_2 = 2022-07-16 20:18:49 UTC`)
  - test: ~Jul 2022 to Sep 2023
- **Columns:** `user_id`, `parent_asin` (the item: a game, accessory or hardware product), `rating`, `timestamp` (milliseconds), `history` (the user's earlier items).
- **The data is not stored in this repo.** Download `Video_Games.train.csv`, `Video_Games.valid.csv` and `Video_Games.test.csv` from the `benchmark/5core/timestamp_w_his` folder of the [Hugging Face dataset](https://huggingface.co/datasets/McAuley-Lab/Amazon-Reviews-2023), and `meta_Video_Games.jsonl` from its `raw/meta_categories` folder, into `data/raw/`.

## Findings

From [`notebooks/01_explore_data.ipynb`](notebooks/01_explore_data.ipynb):

- Files match the dataset page: 736,827 / 34,510 / 43,249 rows (train / valid / test). The cutoffs split cleanly: train before 2021-08-11, valid until 2022-07-16, test until 2023-09-02.
- 94,762 users, 25,612 items, ~815K interactions. No duplicate user–item pairs; every user and item has at least 5 rows.
- A missing `history` marks a user's first review (12.4% of train rows, ~4% of valid and test).
- Users: median 6 rows, mean 8.6, max 473. Most users sit near the 5-review floor, with a few heavy users. In valid/test most users have 1–2 rows; the heaviest has 184.
- Items: median 12 reviews, mean 31.8, max 2,955. The top 1% of items (256) hold 18.6% of all rows.
- Ratings: 64% five-star, 17% four-star, 20% three-star or lower, similar in train and test.
- Cold-start users: 1,416 valid and 2,680 test users never appear in train. By rows, 41% of test rows (32% against train+valid) come from users with no earlier training data.
- Cold-start items: 27.9% of valid rows and 60.5% of test rows involve an item not in train. Even with valid added, 43.0% of test rows (1,542 items) do. An ID-only model can hit at most ~57% of test rows.

From [`notebooks/02_explore_metadata.ipynb`](notebooks/02_explore_metadata.ipynb):

- Every one of the 25,612 items has a metadata row, including the 1,542 items unseen in train+valid.
- `title`, `store` and `categories` are almost always filled. `price` is empty for 33% of items. `description` is empty for 78% of the unseen items, but `features` (bullet points) is filled for 97% of them. `bought_together`, `subtitle` and `author` are empty for every item.
- The catalog is not only games: 40% of items are accessories (70% of the unseen items), and the unseen items skew towards newer hardware (Switch, PS5, Xbox Series, VR).
- `categories` is a hierarchy with inconsistent levels (old consoles get an extra "Legacy Systems" layer), so the model uses all category names, not fixed positions.

## Approach

1. **Baselines.** Most-popular items and a simple "items reviewed together" method, to show the models add value.
2. **Stage 1: retrieval (two-tower + FAISS).** A user tower turns a user's history into a vector and an item tower turns each item's ID and content (title and feature bullets, categories, store, price) into a vector, both trained on how all users behaved. Item vectors are computed once and stored in a FAISS index. At request time the user's vector pulls the closest few hundred candidate items out of ~25.6K.
3. **Stage 2: ranking (LightGBM).** A learning-to-rank model re-orders the candidates using richer features (for example popularity, average rating computed from training data only, price and category from item metadata, and the Stage 1 similarity score) to produce the final list.
4. **Serving.** A FastAPI service packaged in Docker: send a user's history, get back a ranked list of items.

## Evaluation

- Rank the **full catalog** (all ~25.6K items), no sampled negatives.
- Metrics: **Recall@20** and **NDCG@10** on the test period.
- Compare baselines vs. Stage 1 alone vs. Stage 1 + Stage 2.
- Also compare exact search against FAISS (recall lost vs. speed gained).
- Items already in a user's history are removed before ranking.
- Report rows whose item was seen in training separately from rows with new items (43% of test rows involve an item unseen in train+valid).
- Score per row and per user (a few users have up to 184 test rows). Rows with no history (~4% of test) are reported separately.

## Decisions

- **Positives:** ratings of 4 and above count as "liked" (keeps ~80% of rows). Provisional; compare against using all interactions later.
- **Item tower uses content features** (title and feature bullets, all category names, store, price with a missing flag), so items with no training reviews can still be recommended. `description` is not used (empty for 78% of unseen items).
- **Full catalog kept.** Accessories and hardware are part of the dataset and of the test purchases, so they stay in.
- **User tower is built from `history`,** not a per-user ID embedding, because a third of test rows come from users with no earlier training data.
- **Items already in a user's history are excluded from recommendations.** No user reviews the same item twice, so recommending one only wastes a slot.

## Open questions

- **Users with no history.** ~4% of test rows are a user's first-ever review. What does the system recommend for them?
- **Evaluation unit.** Score per row, per user, or both?
- **Heavy users.** Cap or down-weight users with hundreds of reviews in training?
- **Reporting split.** Report results for games and accessories separately? The accessory flag ("Accessories" in categories) is only a rough split.
- **Metadata leakage.** Item rating statistics in the metadata include reviews up to Sep 2023. Compute them from the training period instead.

## Limitations

- 5-core keeps only active users and items with enough reviews, so results look better than a real cold-start setting.
- A review shows interest, not purchase or enjoyment. Offline evaluation can't see items a user would have liked but never reviewed.
- Reviewers are a self-selected group, so conclusions apply to people who review.

## Progress

- [x] Dataset downloaded, repo and environment set up
- [x] Data exploration
- [x] Item metadata check
- [ ] Evaluation setup
- [ ] Baselines
- [ ] Stage 1: two-tower + FAISS
- [ ] Stage 2: LightGBM ranker
- [ ] FastAPI + Docker

## Dataset citation

Hou, Y., Li, J., He, Z., Yan, A., Chen, X., McAuley, J. (2024). _Bridging Language and Items for Retrieval and Recommendation._ arXiv:2403.03952.
