# Failed Session Round-by-Round Routing Audit

- Failed sessions: 18
- override miss: 5
- Cause distribution: final_rank_over_10=15, not_recalled=3

| sample_id | scenario | Cause | Best BM25 | Best Dense | Best Fused | Best Final |
|---|---|---|---:|---:|---:|---:|
| public_0034 | intent_override | final_rank_over_10 | 87 | 112 | 116 | 58 |
| public_0064 | intent_override | final_rank_over_10 | 32 | - | 21 | 11 |
| public_0080 | intent_override | final_rank_over_10 | 29 | - | 28 | 26 |
| public_0144 | intent_override | final_rank_over_10 | 98 | - | 95 | 30 |
| public_0198 | intent_override | final_rank_over_10 | 117 | - | 100 | 36 |
| public_0016 | browsing | final_rank_over_10 | 80 | 102 | 49 | 28 |
| public_0020 | buying | not_recalled | - | - | - | - |
| public_0026 | buying | final_rank_over_10 | 28 | 67 | 26 | 14 |
| public_0028 | buying | final_rank_over_10 | 95 | - | 74 | 31 |
| public_0087 | browsing | final_rank_over_10 | 54 | - | 44 | 53 |
| public_0094 | buying | not_recalled | - | - | - | - |
| public_0126 | browsing | final_rank_over_10 | 37 | - | 40 | 13 |
| public_0145 | buying | final_rank_over_10 | 38 | 67 | 40 | 12 |
| public_0151 | browsing | final_rank_over_10 | 20 | - | 18 | 16 |
| public_0161 | buying | final_rank_over_10 | 28 | - | 23 | 26 |
| public_0174 | buying | final_rank_over_10 | 80 | - | 73 | 28 |
| public_0187 | boundary | final_rank_over_10 | - | 25 | 145 | 86 |
| public_0191 | browsing | not_recalled | - | - | - | - |

> `final` denotes the position in the full reranking list; official hits are accepted only within the top 10. Full round-by-round queries, constraints, prompts, and routing ranks are available in the JSON file in the same directory.
