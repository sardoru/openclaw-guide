# OpenClaw Production Upgrades — Phase 2

## Phase 2: Vector Memory (Semantic Search)

This gives your agent the ability to semantically search all memory files — not just grep, but *meaning-based* retrieval.

### 2.1 Setup

```bash
cd tools/vector-memory
python3 -m venv venv
source venv/bin/activate
pip install faiss-cpu sentence-transformers numpy
```

### 2.2 Indexer — `tools/vector-memory/ingest.py`

```python
#!/usr/bin/env python3
"""
Vector Memory Indexer — FAISS + sentence-transformers
Scans memory files and builds a semantic search index.
"""

import os, json, time, hashlib
from pathlib import Path
from dataclasses import dataclass, asdict
import numpy as np
import faiss
from sentence_transformers import SentenceTransformer

@dataclass
class ChunkMetadata:
    text: str
    source_file: str
    start_line: int
    end_line: int
    timestamp: float
    file_hash: str
    chunk_type: str  # 'section', 'paragraph', 'full'

class MemoryIndexer:
    def __init__(self, model_name='all-MiniLM-L6-v2'):
        self.model = SentenceTransformer(model_name)
        self.dimension = 384
        self.index = faiss.IndexFlatIP(self.dimension)  # Inner product for cosine sim
        self.metadata = []
        self.index_dir = Path('./index')
        self.index_dir.mkdir(exist_ok=True)
        self.hash_file = self.index_dir / 'file_hashes.json'

    def _file_hash(self, filepath):
        with open(filepath, 'rb') as f:
            return hashlib.md5(f.read()).hexdigest()

    def _chunk_markdown(self, text, source_file):
        """Split markdown into sections by headers."""
        chunks = []
        current_chunk = []
        current_start = 1
        
        for i, line in enumerate(text.split('\n'), 1):
            if line.startswith('#') and current_chunk:
                chunks.append(ChunkMetadata(
                    text='\n'.join(current_chunk),
                    source_file=source_file,
                    start_line=current_start,
                    end_line=i - 1,
                    timestamp=time.time(),
                    file_hash=self._file_hash(source_file) if os.path.exists(source_file) else '',
                    chunk_type='section'
                ))
                current_chunk = [line]
                current_start = i
            else:
                current_chunk.append(line)
        
        if current_chunk:
            chunks.append(ChunkMetadata(
                text='\n'.join(current_chunk),
                source_file=source_file,
                start_line=current_start,
                end_line=current_start + len(current_chunk) - 1,
                timestamp=time.time(),
                file_hash='',
                chunk_type='section'
            ))
        return chunks

    def scan_and_index(self, memory_dir):
        """Scan all .md files and build the index."""
        # Load existing hashes for incremental indexing
        old_hashes = {}
        if self.hash_file.exists():
            old_hashes = json.loads(self.hash_file.read_text())

        all_chunks = []
        new_hashes = {}

        for root, _, files in os.walk(memory_dir):
            for f in files:
                if not f.endswith('.md'):
                    continue
                filepath = os.path.join(root, f)
                fhash = self._file_hash(filepath)
                new_hashes[filepath] = fhash
                
                with open(filepath, 'r') as fh:
                    text = fh.read()
                if len(text.strip()) < 50:
                    continue
                
                chunks = self._chunk_markdown(text, filepath)
                # Filter out tiny chunks
                all_chunks.extend([c for c in chunks if len(c.text.strip()) > 30])

        if not all_chunks:
            print("No chunks found.")
            return

        # Encode all chunks
        texts = [c.text for c in all_chunks]
        embeddings = self.model.encode(texts, normalize_embeddings=True, show_progress_bar=True)
        
        self.index = faiss.IndexFlatIP(self.dimension)
        self.index.add(np.array(embeddings).astype('float32'))
        self.metadata = [asdict(c) for c in all_chunks]

        # Save
        faiss.write_index(self.index, str(self.index_dir / 'memory.faiss'))
        with open(self.index_dir / 'metadata.json', 'w') as f:
            json.dump(self.metadata, f)
        with open(self.hash_file, 'w') as f:
            json.dump(new_hashes, f)

        print(f"✓ Indexed {len(all_chunks)} chunks from {len(new_hashes)} files")

if __name__ == '__main__':
    WORKSPACE = os.environ.get('OPENCLAW_WORKSPACE', '/path/to/workspace')  # ← Change
    indexer = MemoryIndexer()
    indexer.scan_and_index(os.path.join(WORKSPACE, 'memory'))
```

### 2.3 Searcher — `tools/vector-memory/search.py`

```python
#!/usr/bin/env python3
"""
Vector Memory Search — semantic search through memory using FAISS.
"""

import os, json, sys
from pathlib import Path
import numpy as np
import faiss
from sentence_transformers import SentenceTransformer

class MemorySearcher:
    def __init__(self, model_name='all-MiniLM-L6-v2'):
        self.model = SentenceTransformer(model_name)
        self.index_dir = Path('./index')
        self.index = faiss.read_index(str(self.index_dir / 'memory.faiss'))
        with open(self.index_dir / 'metadata.json') as f:
            self.metadata = json.load(f)

    def search(self, query, top_k=5):
        embedding = self.model.encode([query], normalize_embeddings=True)
        scores, indices = self.index.search(np.array(embedding).astype('float32'), top_k)
        
        results = []
        for score, idx in zip(scores[0], indices[0]):
            if idx < 0:
                continue
            meta = self.metadata[idx]
            results.append({
                'score': float(score),
                'source': meta['source_file'],
                'lines': f"{meta['start_line']}-{meta['end_line']}",
                'text': meta['text'][:500]
            })
        return results

if __name__ == '__main__':
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument('query', help='Search query')
    parser.add_argument('--top-k', type=int, default=3)
    args = parser.parse_args()

    searcher = MemorySearcher()
    results = searcher.search(args.query, args.top_k)
    
    for i, r in enumerate(results, 1):
        print(f"### Result {i} (score: {r['score']:.3f})")
        print(f"**Source:** {r['source']} (lines {r['lines']})")
        print(f"{r['text']}")
        print()
```

### 2.4 First Index Build

```bash
cd tools/vector-memory
source venv/bin/activate
python3 ingest.py
```

### 2.5 Add to AGENTS.md

Add this to your agent's boot sequence:

```markdown
## Semantic Recall
If the user's first message has a clear topic, run:
cd /path/to/workspace/tools/vector-memory && source venv/bin/activate && python3 search.py "TOPIC" --top-k 3
```

---
