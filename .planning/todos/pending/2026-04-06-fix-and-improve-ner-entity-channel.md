---
created: 2026-04-06T21:10:00.000Z
title: Fix and improve NER entity channel
area: api
files:
  - backend/app/services/retrieval/ner_channel.py
  - backend/app/services/retrieval/engine.py
  - backend/config/default.yaml
---

## Problem

The NER entity retrieval channel (4th channel added in Phase 7) is not working effectively:

1. **Slow**: NER channel Cypher took 3462ms and returned 0 results on "PNRR" query
2. **No results**: The entity detection regex in engine.py may not match parliamentary entities correctly
3. **lawRefs/personRefs empty**: Chunk nodes may not have NER data populated (Phase 4 ENR-03/ENR-04 were marked Pending in requirements)
4. **RRF weight**: `ner_weight: 0.9` in default.yaml but channel contributes nothing

From pipeline run log:
```
NER channel Cypher: 3462.4ms, 0 results
Channels complete: dense=21, sparse=12, graph=0, ner=0
```

## Solution

1. Verify lawRefs/personRefs are actually populated on Chunk nodes in the DB
2. If not populated, run NER extraction on existing chunks (spaCy it_core_news_lg)
3. Optimize the NER channel Cypher query (add index on lawRefs/personRefs arrays)
4. Improve entity detection regex in engine.py to handle Italian parliamentary entity patterns
5. Add a Neo4j array index for efficient CONTAINS queries on lawRefs/personRefs
6. Consider disabling NER channel until data is populated to avoid wasting 3.5s per query
