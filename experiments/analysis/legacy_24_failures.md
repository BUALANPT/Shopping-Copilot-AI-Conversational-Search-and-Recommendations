# Failed Session Round-by-Round Routing Audit

- Failed sessions: 24
- override miss: 7
- Cause distribution: constraint_filter_drop=1, final_rank_over_10=20, not_recalled=3

| sample_id | scenario | Cause | Best BM25 | Best Dense | Best Fused | Best Final |
|---|---|---|---:|---:|---:|---:|
| public_0002 | intent_override | final_rank_over_10 | 65 | - | 80 | 26 |
| public_0003 | intent_override | final_rank_over_10 | - | 90 | 212 | 117 |
| public_0034 | intent_override | final_rank_over_10 | 87 | 112 | 112 | 47 |
| public_0080 | intent_override | final_rank_over_10 | 29 | - | 34 | 28 |
| public_0144 | intent_override | final_rank_over_10 | 98 | - | 96 | 32 |
| public_0197 | intent_override | constraint_filter_drop | 1 | 1 | 1 | - |
| public_0198 | intent_override | final_rank_over_10 | 117 | - | 100 | 42 |
| public_0012 | browsing | final_rank_over_10 | 7 | - | 26 | 20 |
| public_0020 | buying | not_recalled | - | - | - | - |
| public_0028 | buying | final_rank_over_10 | 95 | - | 81 | 36 |
| public_0040 | browsing | final_rank_over_10 | 46 | - | 22 | 13 |
| public_0081 | browsing | final_rank_over_10 | 4 | 112 | 13 | 11 |
| public_0083 | buying | final_rank_over_10 | 98 | 85 | 107 | 36 |
| public_0087 | browsing | final_rank_over_10 | 54 | - | 73 | 81 |
| public_0094 | buying | not_recalled | - | - | - | - |
| public_0099 | browsing | final_rank_over_10 | 4 | - | 16 | 11 |
| public_0120 | browsing | final_rank_over_10 | 94 | - | 119 | 56 |
| public_0126 | browsing | final_rank_over_10 | 37 | - | 67 | 31 |
| public_0151 | browsing | final_rank_over_10 | 20 | - | 25 | 19 |
| public_0161 | buying | final_rank_over_10 | 28 | - | 27 | 29 |
| public_0174 | buying | final_rank_over_10 | 80 | - | 73 | 31 |
| public_0180 | boundary | not_recalled | - | - | - | - |
| public_0187 | boundary | final_rank_over_10 | - | 25 | 76 | 35 |
| public_0195 | browsing | final_rank_over_10 | 5 | - | 29 | 14 |

> `final` denotes the position in the complete reranking list; official hits are accepted only within the top 10. Full round-by-round queries, constraints, prompts, and all routing ranks are available in the JSON files in the same directory.
