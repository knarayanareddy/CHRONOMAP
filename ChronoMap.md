🧠 CHRONOMAP
Comprehensive Engineering Design Document

Version 1.0 | Local-First, Self-Building Personal Knowledge Graph + Temporal Mind Mirror
TABLE OF CONTENTS

    Project Overview & Vision
    Goals, Non-Goals & Constraints
    System Architecture
    Module Breakdown
        4.1 Ingestion Layer
        4.2 Extraction & NLP Layer
        4.3 Graph Builder & Merger
        4.4 Temporal Engine
        4.5 Vector Search Layer
        4.6 Insight Engine
        4.7 Watch Daemon
        4.8 REST API Server
        4.9 Graph UI (Svelte + Three.js + D3)
    Data Models & Schemas
    API Specifications
    Directory Structure
    Configuration System
    Ingestor Deep Dive
    Extraction Pipeline Deep Dive
    Graph Layer Deep Dive
    Temporal Layer Deep Dive
    Insight Engine Deep Dive
    UI & Visualization Deep Dive
    Search Deep Dive
    Privacy & Security Model
    Storage & Persistence
    Logging, Observability & Debugging
    Testing Strategy
    Build, Packaging & Installation
    Platform Support Matrix
    Performance Targets & Benchmarks
    Error Handling Strategy
    Dependency Registry
    Milestone & Phased Rollout Plan
    Open Questions & Future Work

1. Project Overview & Vision
1.1 What is ChronoMap?

ChronoMap is a local-first, passively self-building knowledge graph of everything you touch on your computer. It runs entirely on the user's machine, ingesting files, notes, PDFs, browser history, email archives, source code, ebook highlights, and calendar data — then automatically extracting entities, concepts, and relationships using local NLP — and rendering them as an interactive, glowing 3D neural map.

ChronoMap is:

    A passive knowledge archaeologist: it watches your filesystem and continuously builds your graph without you doing anything
    A temporal mind mirror: it knows not just what you know, but when you started knowing it and how it evolved
    A blind spot detector: it finds concepts you instinctively connect in your behavior but have never explicitly linked
    A semantic search engine over your entire intellectual life
    100% local: no cloud, no AI company, no subscriptions, no data leaving your machine

1.2 The Problem Being Solved

Modern knowledge workers produce enormous amounts of intellectual output across dozens of disconnected tools:

    Notes in Obsidian or Notion
    Highlights in Kindle or Readwise
    PDFs and papers scattered across Downloads
    Browser bookmarks that are never revisited
    Emails where important ideas arrive and die
    Code comments where insights live embedded in logic
    Calendar entries encoding what you prioritized

The result is knowledge amnesia: ideas you once deeply engaged with become invisible. Connections that would spark breakthroughs stay unmade because your tools only visualize what you explicitly linked.

Tools like Obsidian, Roam, and Logseq have graph views — but those graphs only show explicit links ([[like this]]). They require the user to do the linking work. Most people never do.

ChronoMap's bet: your real knowledge graph is implicit, scattered, and temporal. The connections are already there — in co-occurrence, semantic similarity, shared vocabulary, and behavioral patterns across time. ChronoMap makes them visible automatically.
1.3 Design Philosophy
Principle	Description
Passive by default	ChronoMap builds your graph in the background without requiring any manual linking or tagging
Local-first	All NLP, embedding, graph computation, and storage runs on the user's machine
Temporal	Every node and edge carries timestamps; the graph can be replayed at any point in the past
Transparent	Every edge has provenance: which documents created it, with what confidence
Insight-oriented	The system actively surfaces what is hidden: blind spots, islands, obsessions, forgotten ideas
Privacy by design	No telemetry, no cloud model calls, no external API dependencies
Composable	Ingestors, extractors, and insight detectors are pluggable modules
2. Goals, Non-Goals & Constraints
2.1 Goals (In Scope)

    Automatic ingestion from: Markdown notes, PDFs, browser history/bookmarks, local email archives (mbox/maildir), source code, ebook highlights (Kindle/Kobo exports), iCal/ics calendar files
    Local NLP extraction: named entity recognition, keyword/concept extraction, relation extraction, topic modeling, sentence embeddings
    Knowledge graph construction: nodes (entities/concepts), edges (relations/co-occurrence), weights (frequency + semantic similarity)
    Entity deduplication and canonical merging
    Temporal graph: all nodes and edges timestamped, graph_at(t) querying
    Vector-based semantic search over the full graph
    Insight detectors: blind spots, knowledge islands, obsession tracking, forgotten ideas
    Interactive 3D/2D graph visualization with force simulation
    Timeline slider for graph time-travel
    Node detail panel with provenance (source documents)
    Cluster view (topic-level zoom out)
    Semantic search bar with ranked results
    REST API backend
    Watch daemon for continuous filesystem monitoring
    Cross-platform: macOS, Linux, Windows

2.2 Non-Goals (Explicitly Out of Scope)

    ❌ Any cloud sync, remote model API, or telemetry
    ❌ Editing your notes or files (read-only ingestion)
    ❌ Replacing Obsidian/Roam/Logseq as a note-taking tool
    ❌ Real-time collaboration or multi-user support
    ❌ Browser extension (separate concern)
    ❌ Mobile app
    ❌ OCR of scanned documents (Phase 2+)
    ❌ Audio/video ingestion (Phase 2+)
    ❌ Automatic relation extraction with typed predicates (Phase 2 — Phase 1 uses co-occurrence only)

2.3 Constraints

    All NLP and embedding inference must run locally (spaCy, KeyBERT, SentenceTransformers, BERTopic)
    Graph must remain queryable during background re-ingestion (no full lock)
    Vector search must return results in < 200ms for corpora up to 100K chunks
    UI must render graphs of up to 10,000 nodes smoothly (LOD + clustering required)
    Watch daemon overhead: < 1% CPU idle, < 50MB RAM
    SQLite as primary storage (single-file, portable, no server required)
    Python backend (FastAPI) with optional Go wrapper for packaging
    Svelte frontend bundled and served from the backend binary

3. System Architecture
3.1 High-Level Architecture Diagram

text

┌──────────────────────────────────────────────────────────────────────┐
│                         USER'S MACHINE                               │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                    CHRONOMAP CORE                             │   │
│  │                                                               │   │
│  │  ┌─────────────┐    ┌─────────────────────────────────────┐  │   │
│  │  │ Watch Daemon│    │        Ingestion Layer               │  │   │
│  │  │ (watchdog)  │───►│  MD | PDF | Browser | Email | Code  │  │   │
│  │  │             │    │  Highlights | Calendar               │  │   │
│  │  └─────────────┘    └──────────────┬──────────────────────┘  │   │
│  │                                    │                          │   │
│  │                     ┌──────────────▼──────────────────────┐  │   │
│  │                     │       Extraction Layer               │  │   │
│  │                     │  spaCy NER | KeyBERT | BERTopic      │  │   │
│  │                     │  SentenceTransformers | DepParse      │  │   │
│  │                     └──────────────┬──────────────────────┘  │   │
│  │                                    │                          │   │
│  │  ┌──────────────┐  ┌───────────────▼──────────────────────┐  │   │
│  │  │  Vector      │  │         Graph Layer                  │  │   │
│  │  │  Store       │◄─┤  builder | merger | scorer |         │  │   │
│  │  │ (sqlite-vec) │  │  temporal | community detection      │  │   │
│  │  └──────┬───────┘  └───────────────┬──────────────────────┘  │   │
│  │         │                          │                          │   │
│  │  ┌──────▼───────────────────────────▼──────────────────────┐  │   │
│  │  │                  Insight Engine                          │  │   │
│  │  │  BlindSpot | Island | Obsession | ForgottenIdea         │  │   │
│  │  └──────────────────────┬──────────────────────────────────┘  │   │
│  │                         │                                     │   │
│  │  ┌──────────────────────▼──────────────────────────────────┐  │   │
│  │  │              FastAPI REST Server (:8000)                 │  │   │
│  │  └──────────────────────┬──────────────────────────────────┘  │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                             │                                         │
│  ┌──────────────────────────▼──────────────────────────────────────┐ │
│  │             Svelte UI (served at localhost:5173)                 │ │
│  │   GraphCanvas | TimeSlider | NodeDetail | SearchBar |           │ │
│  │   BlindSpotPanel | ClusterView | ObsessionTracker               │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                    SQLite Database                               │ │
│  │   documents | chunks | entities | mentions | edges | embeddings  │ │
│  │   topics | snapshots | insights | settings                       │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘

3.2 Data Flow (Step by Step)

text

Step 1:  Watch daemon detects new/changed file on filesystem
Step 2:  File path queued to ingestion scheduler
Step 3:  Appropriate ingestor reads file → produces normalized Document record
          { doc_id, source_type, uri, title, text_blocks[], created_at, modified_at }
Step 4:  Document chunked (512 token sliding window, 64 token overlap)
Step 5:  Each chunk → SentenceTransformers embedder → 384-dim vector
Step 6:  Chunk vectors upserted to sqlite-vec index
Step 7:  spaCy NER run over chunk → entity mentions extracted
Step 8:  KeyBERT run over full document → top-10 keyphrases extracted
Step 9:  BERTopic assigns topic cluster to document
Step 10: Relation extractor (dep parse) identifies (subject, predicate, object) triples
Step 11: Graph builder:
          - Upsert entity/concept nodes (with first_seen timestamp)
          - Upsert edges (co-occurrence within doc, relation triples)
          - Update edge weights (frequency + semantic similarity score)
          - Update last_seen on all touched nodes/edges
Step 12: Merger runs deduplication pass over new nodes
Step 13: Temporal snapshot recorded (lightweight diff, not full copy)
Step 14: Insight engine flags updated nodes for re-evaluation
Step 15: FastAPI signals connected UI clients via WebSocket
Step 16: UI fetches updated subgraph and re-renders

3.3 Component Ownership
Component	Language	Owns
Watch Daemon	Python (watchdog)	Filesystem event monitoring, change queuing
Ingestion Layer	Python	Source-type parsing, document normalization
Extraction Layer	Python (spaCy, KeyBERT, BERTopic, SentenceTransformers)	NLP pipeline, entity/concept/relation/vector extraction
Graph Layer	Python (NetworkX + SQLite)	Graph construction, deduplication, scoring, snapshots
Temporal Engine	Python	Timestamped nodes/edges, graph_at(t) queries
Vector Store	Python (sqlite-vec / Chroma)	kNN semantic search, embedding persistence
Insight Engine	Python	Blind spot detection, island detection, obsession tracking
REST API	Python (FastAPI)	HTTP endpoints, WebSocket push
UI	Svelte + Three.js + D3	Graph rendering, time slider, search, panels
Storage	SQLite	Documents, chunks, entities, edges, embeddings, settings
4. Module Breakdown
4.1 Ingestion Layer

Purpose

The ingestion layer is responsible for reading source files and producing a normalized, source-agnostic Document record that the rest of the pipeline can process uniformly. Each ingestor is an independent module that handles one source type.

Ingestor Interface

Python

# ingestors/base.py

from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime

@dataclass
class TextBlock:
    text: str
    block_type: str        # "paragraph" | "heading" | "code" | "highlight" | "metadata"
    offset: int            # Character offset within full document text
    page_number: Optional[int] = None
    metadata: dict = None  # Source-specific metadata (chapter title, URL, etc.)

@dataclass
class Document:
    doc_id: str            # UUID, stable across re-ingests
    source_type: str       # "markdown" | "pdf" | "browser" | "email" | "code" | "highlight" | "calendar"
    uri: str               # Absolute path or URL
    title: str
    author: Optional[str]
    created_at: datetime
    modified_at: datetime
    ingested_at: datetime
    full_text: str
    blocks: List[TextBlock]
    source_metadata: dict  # Source-specific fields (domain, subject, language, etc.)
    checksum: str          # SHA256 of full_text — used to detect changes

class BaseIngestor(ABC):
    @abstractmethod
    def can_handle(self, uri: str) -> bool:
        """Return True if this ingestor handles the given file type or URI."""
        pass

    @abstractmethod
    def ingest(self, uri: str) -> Document:
        """Read the source and return a normalized Document."""
        pass

    @abstractmethod
    def supports_incremental(self) -> bool:
        """Whether this ingestor supports partial re-ingestion."""
        pass

Markdown Ingestor

Python

# ingestors/markdown_ingestor.py

import re
import hashlib
from pathlib import Path
from datetime import datetime
import frontmatter
from .base import BaseIngestor, Document, TextBlock

class MarkdownIngestor(BaseIngestor):
    HEADING_RE = re.compile(r'^(#{1,6})\s+(.+)$', re.MULTILINE)
    LINK_RE    = re.compile(r'\[\[(.+?)(?:\|.+?)?\]\]')  # WikiLinks
    
    def can_handle(self, uri: str) -> bool:
        return Path(uri).suffix.lower() in ('.md', '.markdown', '.mdx')

    def ingest(self, uri: str) -> Document:
        path = Path(uri)
        post = frontmatter.load(uri)
        raw_text = post.content
        stat = path.stat()

        # Extract frontmatter dates if present
        created_at = post.metadata.get('created', datetime.fromtimestamp(stat.st_ctime))
        modified_at = post.metadata.get('modified', datetime.fromtimestamp(stat.st_mtime))

        blocks = self._extract_blocks(raw_text)
        
        # Extract WikiLinks as explicit relation hints
        wikilinks = self.LINK_RE.findall(raw_text)

        return Document(
            doc_id=self._stable_id(uri),
            source_type='markdown',
            uri=str(path.resolve()),
            title=post.metadata.get('title', path.stem),
            author=post.metadata.get('author'),
            created_at=created_at,
            modified_at=modified_at,
            ingested_at=datetime.utcnow(),
            full_text=raw_text,
            blocks=blocks,
            source_metadata={
                'tags': post.metadata.get('tags', []),
                'wikilinks': wikilinks,
                'frontmatter': dict(post.metadata),
            },
            checksum=hashlib.sha256(raw_text.encode()).hexdigest()
        )

    def _extract_blocks(self, text: str) -> list[TextBlock]:
        blocks = []
        # Split on headings and paragraphs
        # ... implementation
        return blocks

    def _stable_id(self, uri: str) -> str:
        """Produce a stable UUID from the file path (doesn't change across re-ingests)."""
        import uuid
        return str(uuid.uuid5(uuid.NAMESPACE_URL, uri))

    def supports_incremental(self) -> bool:
        return False  # Full re-ingest on change

PDF Ingestor

Python

# ingestors/pdf_ingestor.py

import fitz  # PyMuPDF
from pathlib import Path
from datetime import datetime
import hashlib
from .base import BaseIngestor, Document, TextBlock

class PDFIngestor(BaseIngestor):
    def can_handle(self, uri: str) -> bool:
        return Path(uri).suffix.lower() == '.pdf'

    def ingest(self, uri: str) -> Document:
        doc = fitz.open(uri)
        meta = doc.metadata
        
        blocks = []
        full_text_parts = []
        highlights = []
        
        for page_num, page in enumerate(doc, start=1):
            # Extract text blocks with position data
            page_dict = page.get_text("dict")
            for block in page_dict.get("blocks", []):
                if block.get("type") == 0:  # Text block
                    block_text = " ".join(
                        span["text"] for line in block.get("lines", [])
                        for span in line.get("spans", [])
                    ).strip()
                    if block_text:
                        blocks.append(TextBlock(
                            text=block_text,
                            block_type="paragraph",
                            offset=len("\n".join(full_text_parts)),
                            page_number=page_num,
                        ))
                        full_text_parts.append(block_text)

            # Extract annotations (highlights)
            for annot in page.annots():
                if annot.type[0] == 8:  # Highlight annotation
                    highlighted_text = page.get_textbox(annot.rect)
                    if highlighted_text.strip():
                        highlights.append({
                            'text': highlighted_text.strip(),
                            'page': page_num,
                            'color': annot.colors.get('stroke'),
                        })
                        blocks.append(TextBlock(
                            text=highlighted_text.strip(),
                            block_type="highlight",
                            offset=len("\n".join(full_text_parts)),
                            page_number=page_num,
                            metadata={'color': annot.colors.get('stroke')},
                        ))

        full_text = "\n\n".join(full_text_parts)
        stat = Path(uri).stat()

        return Document(
            doc_id=self._stable_id(uri),
            source_type='pdf',
            uri=str(Path(uri).resolve()),
            title=meta.get('title') or Path(uri).stem,
            author=meta.get('author'),
            created_at=self._parse_pdf_date(meta.get('creationDate')) or datetime.fromtimestamp(stat.st_ctime),
            modified_at=self._parse_pdf_date(meta.get('modDate')) or datetime.fromtimestamp(stat.st_mtime),
            ingested_at=datetime.utcnow(),
            full_text=full_text,
            blocks=blocks,
            source_metadata={
                'page_count': len(doc),
                'highlights': highlights,
                'subject': meta.get('subject'),
                'keywords': meta.get('keywords'),
            },
            checksum=hashlib.sha256(full_text.encode()).hexdigest()
        )

    def _parse_pdf_date(self, pdf_date_str: str | None) -> datetime | None:
        if not pdf_date_str:
            return None
        try:
            # PDF date format: D:YYYYMMDDHHmmSS
            clean = pdf_date_str.replace("D:", "")[:14]
            return datetime.strptime(clean, "%Y%m%d%H%M%S")
        except Exception:
            return None

    def _stable_id(self, uri: str) -> str:
        import uuid
        return str(uuid.uuid5(uuid.NAMESPACE_URL, uri))

    def supports_incremental(self) -> bool:
        return False

Browser History Ingestor

Python

# ingestors/browser_ingestor.py

import sqlite3
import shutil
import tempfile
from pathlib import Path
from datetime import datetime, timezone
from .base import BaseIngestor, Document, TextBlock

class BrowserHistoryIngestor(BaseIngestor):
    # Chrome/Chromium history DB path per platform
    CHROME_PATHS = {
        'darwin':  Path.home() / 'Library/Application Support/Google/Chrome/Default/History',
        'linux':   Path.home() / '.config/google-chrome/Default/History',
        'windows': Path.home() / 'AppData/Local/Google/Chrome/User Data/Default/History',
    }

    def can_handle(self, uri: str) -> bool:
        return uri.startswith('browser://') or 'History' in uri

    def ingest(self, uri: str) -> Document:
        import sys
        db_path = self.CHROME_PATHS.get(sys.platform, self.CHROME_PATHS['linux'])
        
        if not db_path.exists():
            raise FileNotFoundError(f"Chrome history not found at {db_path}")

        # Copy DB to temp location — Chrome locks the live DB
        with tempfile.NamedTemporaryFile(suffix='.db', delete=False) as tmp:
            shutil.copy2(db_path, tmp.name)
            conn = sqlite3.connect(tmp.name)
            cursor = conn.cursor()

            # Chrome stores time as microseconds since 1601-01-01
            EPOCH_OFFSET = 11644473600  # seconds between 1601-01-01 and 1970-01-01
            
            cursor.execute("""
                SELECT url, title, visit_count, last_visit_time
                FROM urls
                WHERE hidden = 0
                ORDER BY last_visit_time DESC
                LIMIT 10000
            """)
            rows = cursor.fetchall()
            conn.close()

        blocks = []
        full_text_parts = []
        
        for url, title, visit_count, last_visit_chrome in rows:
            last_visit_unix = (last_visit_chrome / 1_000_000) - EPOCH_OFFSET
            block_text = f"{title or 'Untitled'}: {url}"
            blocks.append(TextBlock(
                text=block_text,
                block_type='metadata',
                offset=len('\n'.join(full_text_parts)),
                metadata={
                    'url': url,
                    'title': title,
                    'visit_count': visit_count,
                    'last_visit': datetime.fromtimestamp(last_visit_unix, tz=timezone.utc).isoformat(),
                }
            ))
            full_text_parts.append(block_text)

        full_text = '\n'.join(full_text_parts)

        return Document(
            doc_id=self._stable_id('browser://chrome/history'),
            source_type='browser',
            uri='browser://chrome/history',
            title='Chrome Browser History',
            author=None,
            created_at=datetime.utcnow(),
            modified_at=datetime.utcnow(),
            ingested_at=datetime.utcnow(),
            full_text=full_text,
            blocks=blocks,
            source_metadata={'browser': 'chrome', 'entry_count': len(rows)},
            checksum=self._checksum(full_text),
        )

    def _stable_id(self, uri): 
        import uuid
        return str(uuid.uuid5(uuid.NAMESPACE_URL, uri))
    
    def _checksum(self, text):
        import hashlib
        return hashlib.sha256(text.encode()).hexdigest()

    def supports_incremental(self):
        return True  # Can filter by date on subsequent runs

Ingestor Registry

Python

# ingestors/registry.py

from .markdown_ingestor import MarkdownIngestor
from .pdf_ingestor import PDFIngestor
from .browser_ingestor import BrowserHistoryIngestor
from .email_ingestor import EmailIngestor
from .code_ingestor import CodeIngestor
from .highlight_ingestor import HighlightIngestor
from .calendar_ingestor import CalendarIngestor

class IngestorRegistry:
    def __init__(self):
        self._ingestors = [
            MarkdownIngestor(),
            PDFIngestor(),
            BrowserHistoryIngestor(),
            EmailIngestor(),
            CodeIngestor(),
            HighlightIngestor(),
            CalendarIngestor(),
        ]

    def get_ingestor(self, uri: str):
        for ingestor in self._ingestors:
            if ingestor.can_handle(uri):
                return ingestor
        raise ValueError(f"No ingestor found for: {uri}")

    def register(self, ingestor):
        """Allow third-party ingestors to be registered at runtime."""
        self._ingestors.insert(0, ingestor)  # User-registered take priority

4.2 Extraction & NLP Layer

Purpose

The extraction layer takes a normalized Document and runs multiple NLP passes to produce:

    Named entity mentions (persons, places, organizations, dates, events)
    Concept/keyword mentions (key phrases and topics)
    Sentence-level embeddings (for semantic similarity and search)
    Topic assignments (BERTopic cluster IDs)
    Relation triples (subject–predicate–object from dependency parsing)

Chunker

Python

# extractor/chunker.py

from dataclasses import dataclass
from typing import List

@dataclass
class Chunk:
    chunk_id: str
    doc_id: str
    text: str
    token_start: int   # Token index in document
    token_end: int
    char_start: int    # Char offset in document full_text
    char_end: int
    source_block_ids: List[str]  # Which TextBlocks contributed to this chunk

class Chunker:
    def __init__(self, chunk_size: int = 512, overlap: int = 64):
        self.chunk_size = chunk_size
        self.overlap = overlap

    def chunk(self, doc) -> List[Chunk]:
        """
        Sliding window chunking over document full_text.
        Respects sentence boundaries where possible using spaCy sentence segmentation.
        """
        import spacy
        nlp = spacy.load("en_core_web_sm", disable=["ner", "lemmatizer"])
        spacy_doc = nlp(doc.full_text[:1_000_000])  # 1MB safety cap
        
        sentences = list(spacy_doc.sents)
        chunks = []
        current_tokens = []
        current_sents = []
        
        for sent in sentences:
            sent_tokens = [t for t in sent]
            if len(current_tokens) + len(sent_tokens) > self.chunk_size:
                if current_tokens:
                    chunk_text = "".join([t.text_with_ws for t in current_tokens])
                    chunks.append(self._make_chunk(doc.doc_id, chunk_text, current_tokens))
                    # Overlap: keep last `overlap` tokens
                    current_tokens = current_tokens[-self.overlap:]
            current_tokens.extend(sent_tokens)
        
        if current_tokens:
            chunk_text = "".join([t.text_with_ws for t in current_tokens])
            chunks.append(self._make_chunk(doc.doc_id, chunk_text, current_tokens))

        return chunks

    def _make_chunk(self, doc_id, text, tokens) -> Chunk:
        import uuid, hashlib
        chunk_id = str(uuid.uuid5(
            uuid.NAMESPACE_URL,
            f"{doc_id}:{tokens[0].idx}:{tokens[-1].idx}"
        ))
        return Chunk(
            chunk_id=chunk_id,
            doc_id=doc_id,
            text=text.strip(),
            token_start=tokens[0].i,
            token_end=tokens[-1].i,
            char_start=tokens[0].idx,
            char_end=tokens[-1].idx + len(tokens[-1].text),
            source_block_ids=[],
        )

Entity Extractor

Python

# extractor/entity_extractor.py

import spacy
from dataclasses import dataclass
from typing import List

@dataclass
class EntityMention:
    mention_id: str
    doc_id: str
    chunk_id: str
    text: str           # Surface form ("Alan Turing")
    label: str          # spaCy NER label: PERSON, ORG, GPE, DATE, EVENT, etc.
    start_char: int     # Char offset in chunk
    end_char: int
    confidence: float   # spaCy NER confidence (if available)

class EntityExtractor:
    # Labels we care about — filter noise
    ALLOWED_LABELS = {
        'PERSON', 'ORG', 'GPE', 'LOC', 'PRODUCT', 'EVENT',
        'WORK_OF_ART', 'LAW', 'LANGUAGE', 'NORP', 'FAC',
    }

    def __init__(self, model: str = "en_core_web_trf"):
        # en_core_web_trf for accuracy; en_core_web_sm for speed
        self.nlp = spacy.load(model)

    def extract(self, chunk) -> List[EntityMention]:
        doc = self.nlp(chunk.text)
        mentions = []
        for ent in doc.ents:
            if ent.label_ not in self.ALLOWED_LABELS:
                continue
            if len(ent.text.strip()) < 2:
                continue
            mentions.append(EntityMention(
                mention_id=f"{chunk.chunk_id}:{ent.start_char}:{ent.end_char}",
                doc_id=chunk.doc_id,
                chunk_id=chunk.chunk_id,
                text=ent.text.strip(),
                label=ent.label_,
                start_char=ent.start_char,
                end_char=ent.end_char,
                confidence=ent._.score if hasattr(ent._, 'score') else 1.0,
            ))
        return mentions

Concept Extractor (KeyBERT)

Python

# extractor/concept_extractor.py

from keybert import KeyBERT
from dataclasses import dataclass
from typing import List

@dataclass
class ConceptMention:
    mention_id: str
    doc_id: str
    phrase: str
    score: float       # Relevance score from KeyBERT (0.0-1.0)
    source: str        # "keybert" | "tfidf_fallback"

class ConceptExtractor:
    def __init__(self, model_name: str = "all-MiniLM-L6-v2"):
        self.kw_model = KeyBERT(model=model_name)

    def extract(self, doc, top_n: int = 10) -> List[ConceptMention]:
        keywords = self.kw_model.extract_keywords(
            doc.full_text,
            keyphrase_ngram_range=(1, 3),
            stop_words='english',
            top_n=top_n,
            use_maxsum=True,     # Reduce redundancy
            nr_candidates=20,
        )
        mentions = []
        for phrase, score in keywords:
            if score < 0.15:  # Low-signal threshold
                continue
            mentions.append(ConceptMention(
                mention_id=f"{doc.doc_id}:concept:{phrase}",
                doc_id=doc.doc_id,
                phrase=phrase.lower().strip(),
                score=score,
                source='keybert',
            ))
        return mentions

Embedder (SentenceTransformers)

Python

# extractor/embedder.py

from sentence_transformers import SentenceTransformer
from typing import List
import numpy as np

class Embedder:
    MODEL_NAME = "sentence-transformers/all-MiniLM-L6-v2"
    EMBEDDING_DIM = 384

    def __init__(self):
        self.model = SentenceTransformer(self.MODEL_NAME)

    def embed_chunks(self, chunks: List) -> List[np.ndarray]:
        """Batch-embed a list of Chunk objects. Returns list of 384-dim vectors."""
        texts = [c.text for c in chunks]
        return self.model.encode(
            texts,
            batch_size=32,
            show_progress_bar=False,
            normalize_embeddings=True,  # Unit vectors for cosine similarity via dot product
        )

    def embed_query(self, query: str) -> np.ndarray:
        """Embed a search query string."""
        return self.model.encode(
            query,
            normalize_embeddings=True,
        )

    def embed_node(self, node_text: str) -> np.ndarray:
        """Embed a node's canonical label for blind spot detection."""
        return self.model.encode(node_text, normalize_embeddings=True)

Topic Modeler (BERTopic)

Python

# extractor/topic_modeler.py

from bertopic import BERTopic
from sentence_transformers import SentenceTransformer
from umap import UMAP
from hdbscan import HDBSCAN
from typing import List, Dict

class TopicModeler:
    def __init__(self):
        self.model = BERTopic(
            embedding_model=SentenceTransformer("all-MiniLM-L6-v2"),
            umap_model=UMAP(n_neighbors=15, n_components=5, min_dist=0.0, metric='cosine'),
            hdbscan_model=HDBSCAN(min_cluster_size=5, metric='euclidean', prediction_data=True),
            nr_topics='auto',
            calculate_probabilities=False,
        )
        self._fitted = False

    def fit_transform(self, docs: List[str]) -> tuple[List[int], Dict]:
        """
        Fit model on a corpus and return topic assignments.
        Returns: (topic_ids, topic_info_dict)
        """
        topics, _ = self.model.fit_transform(docs)
        self._fitted = True
        topic_info = self.model.get_topic_info().to_dict('records')
        return topics, topic_info

    def transform(self, docs: List[str]) -> List[int]:
        """Assign topics to new documents without refitting."""
        if not self._fitted:
            raise RuntimeError("Model must be fit before transform.")
        topics, _ = self.model.transform(docs)
        return topics

    def get_topic_label(self, topic_id: int) -> str:
        """Return a human-readable label for a topic."""
        if topic_id == -1:
            return "Uncategorized"
        words = self.model.get_topic(topic_id)
        if not words:
            return f"Topic {topic_id}"
        return " / ".join([w for w, _ in words[:3]])

Relation Extractor

Python

# extractor/relation_extractor.py

import spacy
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class RelationTriple:
    triple_id: str
    doc_id: str
    chunk_id: str
    subject_text: str
    predicate_text: str
    object_text: str
    confidence: float
    extraction_method: str  # "dep_parse" | "cooccurrence"

class RelationExtractor:
    def __init__(self):
        self.nlp = spacy.load("en_core_web_sm")

    def extract(self, chunk) -> List[RelationTriple]:
        doc = self.nlp(chunk.text)
        triples = []

        for sent in doc.sents:
            # Find root verb
            root = None
            for token in sent:
                if token.dep_ == "ROOT" and token.pos_ == "VERB":
                    root = token
                    break
            if not root:
                continue

            # Find subject and object
            subj = self._find_dep(root, {"nsubj", "nsubjpass"})
            obj  = self._find_dep(root, {"dobj", "pobj", "attr", "acomp"})

            if subj and obj:
                triples.append(RelationTriple(
                    triple_id=f"{chunk.chunk_id}:{root.i}",
                    doc_id=chunk.doc_id,
                    chunk_id=chunk.chunk_id,
                    subject_text=self._expand_span(subj),
                    predicate_text=root.lemma_.lower(),
                    object_text=self._expand_span(obj),
                    confidence=0.6,  # Dep parse confidence floor
                    extraction_method="dep_parse",
                ))

        return triples

    def _find_dep(self, head, dep_labels):
        for child in head.children:
            if child.dep_ in dep_labels:
                return child
        return None

    def _expand_span(self, token) -> str:
        """Include compound and adjectival modifiers."""
        modifiers = [t for t in token.subtree if t.dep_ in {'compound', 'amod', 'det'}]
        all_tokens = sorted(modifiers + [token], key=lambda t: t.i)
        return " ".join(t.text for t in all_tokens)

4.3 Graph Builder & Merger

Purpose

The graph builder takes extracted entities, concepts, and relations and constructs a persistent knowledge graph. It handles node/edge upserts, edge weight computation, and deduplication merging.

Node and Edge Types

Python

# graph/types.py

from dataclasses import dataclass, field
from datetime import datetime
from typing import List, Optional

@dataclass
class KGNode:
    node_id: str
    label: str              # Canonical surface form
    node_type: str          # "entity" | "concept" | "topic" | "document"
    entity_label: str       # spaCy label if entity: PERSON, ORG, GPE, etc.
    first_seen: datetime
    last_seen: datetime
    mention_count: int
    doc_ids: List[str]      # Documents this node appears in
    embedding: Optional[list] = None  # 384-dim vector of canonical label

@dataclass
class KGEdge:
    edge_id: str
    source_node_id: str
    target_node_id: str
    edge_type: str          # "cooccurrence" | "relation" | "semantic" | "wikilink"
    weight: float           # 0.0-1.0: frequency × semantic_boost
    predicate: Optional[str]  # Verb lemma if from relation extraction
    first_seen: datetime
    last_seen: datetime
    doc_ids: List[str]      # Documents where this edge was observed
    confidence: float

Graph Builder

Python

# graph/builder.py

import networkx as nx
from datetime import datetime
from typing import List
from .types import KGNode, KGEdge
from ..storage.graph_repo import GraphRepository
from ..extractor.entity_extractor import EntityMention
from ..extractor.concept_extractor import ConceptMention
from ..extractor.relation_extractor import RelationTriple
import uuid
import numpy as np

class GraphBuilder:
    def __init__(self, repo: GraphRepository, embedder):
        self.repo = repo
        self.embedder = embedder

    def ingest_document(
        self,
        doc_id: str,
        entity_mentions: List[EntityMention],
        concept_mentions: List[ConceptMention],
        relation_triples: List[RelationTriple],
        timestamp: datetime,
    ):
        now = timestamp

        # 1. Upsert entity nodes
        entity_nodes = {}
        for mention in entity_mentions:
            node_id = self._node_id_for(mention.text, mention.label)
            node = self.repo.get_node(node_id)
            if node:
                node.last_seen = now
                node.mention_count += 1
                if doc_id not in node.doc_ids:
                    node.doc_ids.append(doc_id)
                self.repo.update_node(node)
            else:
                embedding = self.embedder.embed_node(mention.text).tolist()
                node = KGNode(
                    node_id=node_id,
                    label=mention.text,
                    node_type="entity",
                    entity_label=mention.label,
                    first_seen=now,
                    last_seen=now,
                    mention_count=1,
                    doc_ids=[doc_id],
                    embedding=embedding,
                )
                self.repo.insert_node(node)
            entity_nodes[mention.text] = node_id

        # 2. Upsert concept nodes
        concept_nodes = {}
        for mention in concept_mentions:
            node_id = self._node_id_for(mention.phrase, "CONCEPT")
            node = self.repo.get_node(node_id)
            if node:
                node.last_seen = now
                node.mention_count += 1
                if doc_id not in node.doc_ids:
                    node.doc_ids.append(doc_id)
                self.repo.update_node(node)
            else:
                embedding = self.embedder.embed_node(mention.phrase).tolist()
                node = KGNode(
                    node_id=node_id,
                    label=mention.phrase,
                    node_type="concept",
                    entity_label="CONCEPT",
                    first_seen=now,
                    last_seen=now,
                    mention_count=1,
                    doc_ids=[doc_id],
                    embedding=embedding,
                )
                self.repo.insert_node(node)
            concept_nodes[mention.phrase] = node_id

        # 3. Build co-occurrence edges between entities within same document
        all_nodes_in_doc = list(entity_nodes.values()) + list(concept_nodes.values())
        for i, src_id in enumerate(all_nodes_in_doc):
            for tgt_id in all_nodes_in_doc[i+1:]:
                if src_id == tgt_id:
                    continue
                self._upsert_edge(src_id, tgt_id, "cooccurrence", doc_id, now, predicate=None)

        # 4. Relation triple edges
        all_labels = {**entity_nodes, **concept_nodes}
        for triple in relation_triples:
            src_id = all_labels.get(triple.subject_text)
            tgt_id = all_labels.get(triple.object_text)
            if src_id and tgt_id:
                self._upsert_edge(src_id, tgt_id, "relation", doc_id, now, predicate=triple.predicate_text)

    def _upsert_edge(self, src_id, tgt_id, edge_type, doc_id, now, predicate):
        edge_id = self._edge_id(src_id, tgt_id, edge_type)
        edge = self.repo.get_edge(edge_id)
        if edge:
            edge.weight = min(edge.weight + 0.05, 1.0)  # Increment, cap at 1.0
            edge.last_seen = now
            if doc_id not in edge.doc_ids:
                edge.doc_ids.append(doc_id)
            self.repo.update_edge(edge)
        else:
            edge = KGEdge(
                edge_id=edge_id,
                source_node_id=src_id,
                target_node_id=tgt_id,
                edge_type=edge_type,
                weight=0.1,
                predicate=predicate,
                first_seen=now,
                last_seen=now,
                doc_ids=[doc_id],
                confidence=0.7,
            )
            self.repo.insert_edge(edge)

    def _node_id_for(self, text: str, label: str) -> str:
        return str(uuid.uuid5(uuid.NAMESPACE_URL, f"{label.lower()}:{text.lower().strip()}"))

    def _edge_id(self, src_id: str, tgt_id: str, edge_type: str) -> str:
        # Canonical: sort src/tgt so A→B and B→A share an ID (undirected)
        pair = tuple(sorted([src_id, tgt_id]))
        return str(uuid.uuid5(uuid.NAMESPACE_URL, f"{pair[0]}:{pair[1]}:{edge_type}"))

Entity Merger (Deduplication)

Python

# graph/merger.py

from typing import List, Tuple
import numpy as np

class EntityMerger:
    """
    Identifies nodes that refer to the same real-world entity under different
    surface forms and merges them into a single canonical node.
    
    Strategy (layered, in priority order):
    1. Exact string match after normalization
    2. Alias match (known aliases from a lookup table)
    3. Embedding cosine similarity above threshold
    4. Context overlap (share >= 50% of document co-occurrences)
    """
    SIMILARITY_THRESHOLD = 0.92  # High threshold to avoid over-merging

    def __init__(self, repo, vector_store):
        self.repo = repo
        self.vector_store = vector_store

    def find_merge_candidates(self, node) -> List[Tuple[str, str, float]]:
        """
        Returns list of (candidate_node_id, reason, confidence) for nodes
        that should be merged into the given node.
        """
        candidates = []

        # Step 1: Normalize and exact match
        normalized = self._normalize(node.label)
        exact_matches = self.repo.find_nodes_by_normalized_label(normalized)
        for match in exact_matches:
            if match.node_id != node.node_id:
                candidates.append((match.node_id, "exact_normalized", 1.0))

        # Step 2: Embedding kNN (only for entity nodes, not concepts)
        if node.node_type == "entity" and node.embedding:
            neighbors = self.vector_store.query_nodes(
                embedding=node.embedding,
                top_k=5,
                filter_type="entity",
            )
            for neighbor_id, similarity in neighbors:
                if similarity >= self.SIMILARITY_THRESHOLD and neighbor_id != node.node_id:
                    # Verify same entity type before merging
                    neighbor = self.repo.get_node(neighbor_id)
                    if neighbor and neighbor.entity_label == node.entity_label:
                        candidates.append((neighbor_id, "embedding_similarity", similarity))

        return candidates

    def merge(self, primary_node_id: str, secondary_node_id: str):
        """
        Merge secondary into primary: redirect all edges, update doc_ids,
        sum mention counts, delete secondary.
        """
        primary = self.repo.get_node(primary_node_id)
        secondary = self.repo.get_node(secondary_node_id)

        if not primary or not secondary:
            return

        # Merge doc_ids
        primary.doc_ids = list(set(primary.doc_ids + secondary.doc_ids))
        primary.mention_count += secondary.mention_count
        primary.first_seen = min(primary.first_seen, secondary.first_seen)
        primary.last_seen = max(primary.last_seen, secondary.last_seen)

        # Redirect all edges from secondary to primary
        self.repo.redirect_edges(secondary_node_id, primary_node_id)

        # Record merge in aliases table
        self.repo.record_alias(secondary.label, primary_node_id)

        # Delete secondary node
        self.repo.delete_node(secondary_node_id)

        # Update primary
        self.repo.update_node(primary)

    def _normalize(self, text: str) -> str:
        import re
        return re.sub(r'\s+', ' ', text.lower().strip())

4.4 Temporal Engine

Purpose

Every node and edge in ChronoMap carries temporal metadata. The temporal engine provides graph_at(timestamp) queries, manages the snapshot log, and powers the UI time slider.

Python

# graph/temporal.py

from datetime import datetime
from typing import Optional
import networkx as nx

class TemporalGraph:
    """
    Provides time-filtered views of the knowledge graph.
    
    Each node/edge has:
      - first_seen: when it first appeared (document creation/modification date)
      - last_seen:  the most recent document that referenced it
    
    graph_at(t) returns the subgraph that was "alive" at time t:
      - Nodes where first_seen <= t AND (last_seen >= t OR still active)
      - Edges where first_seen <= t AND (last_seen >= t OR still active)
    
    "Still active" = the node/edge was seen within the configured staleness
    window (default: 90 days after last_seen, configurable).
    """

    STALENESS_DAYS = 90  # Days after last_seen before a node is considered "gone"

    def __init__(self, repo):
        self.repo = repo

    def graph_at(self, timestamp: datetime, staleness_days: Optional[int] = None) -> nx.Graph:
        """
        Return a NetworkX graph representing ChronoMap at the given timestamp.
        """
        staleness = staleness_days or self.STALENESS_DAYS
        
        nodes = self.repo.get_nodes_at(timestamp, staleness_days=staleness)
        edges = self.repo.get_edges_at(timestamp, staleness_days=staleness)

        G = nx.Graph()
        for node in nodes:
            G.add_node(
                node.node_id,
                label=node.label,
                node_type=node.node_type,
                entity_label=node.entity_label,
                mention_count=node.mention_count,
                first_seen=node.first_seen.isoformat(),
                last_seen=node.last_seen.isoformat(),
            )
        for edge in edges:
            G.add_edge(
                edge.source_node_id,
                edge.target_node_id,
                edge_id=edge.edge_id,
                edge_type=edge.edge_type,
                weight=edge.weight,
                predicate=edge.predicate,
                first_seen=edge.first_seen.isoformat(),
                last_seen=edge.last_seen.isoformat(),
            )
        return G

    def get_timeline_keypoints(self) -> list[dict]:
        """
        Return a list of significant timestamps for the time slider:
        - First document ever ingested
        - Each month that had significant activity (> 10 new nodes)
        - Most recent update
        """
        return self.repo.get_timeline_keypoints()

    def diff(self, t1: datetime, t2: datetime) -> dict:
        """
        Return nodes and edges added, removed, and changed between t1 and t2.
        Used for animated transitions in the UI.
        """
        g1 = self.graph_at(t1)
        g2 = self.graph_at(t2)
        
        nodes_added   = set(g2.nodes) - set(g1.nodes)
        nodes_removed = set(g1.nodes) - set(g2.nodes)
        edges_added   = set(g2.edges) - set(g1.edges)
        edges_removed = set(g1.edges) - set(g2.edges)

        return {
            'nodes_added':   list(nodes_added),
            'nodes_removed': list(nodes_removed),
            'edges_added':   list(edges_added),
            'edges_removed': list(edges_removed),
            'summary': f"+{len(nodes_added)} nodes, -{len(nodes_removed)} nodes",
        }

4.5 Vector Search Layer

Purpose

The vector store persists chunk embeddings and node embeddings, providing sub-200ms kNN search for semantic search and blind spot detection.

Python

# vector_store/store.py

from abc import ABC, abstractmethod
from typing import List, Tuple
import numpy as np

class VectorStore(ABC):
    @abstractmethod
    def upsert_chunk(self, chunk_id: str, doc_id: str, embedding: np.ndarray, text: str, metadata: dict): pass

    @abstractmethod
    def upsert_node(self, node_id: str, embedding: np.ndarray, label: str, node_type: str): pass

    @abstractmethod
    def query_chunks(self, embedding: np.ndarray, top_k: int, filter_doc_ids: List[str] = None) -> List[Tuple[str, str, float]]:
        """Returns list of (chunk_id, doc_id, score)"""
        pass

    @abstractmethod
    def query_nodes(self, embedding: np.ndarray, top_k: int, filter_type: str = None) -> List[Tuple[str, float]]:
        """Returns list of (node_id, score)"""
        pass

    @abstractmethod
    def delete_by_doc(self, doc_id: str): pass

SQLite-Vec Backend

Python

# vector_store/sqlite_vec_store.py

import sqlite_vec
import sqlite3
import struct
import numpy as np
from typing import List, Tuple, Optional
from .store import VectorStore

class SqliteVecStore(VectorStore):
    """
    Uses sqlite-vec for ANN (approximate nearest neighbor) search
    directly inside the SQLite database. No external vector DB required.
    """
    EMBEDDING_DIM = 384

    def __init__(self, db_path: str):
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self.conn.enable_load_extension(True)
        sqlite_vec.load(self.conn)
        self.conn.enable_load_extension(False)
        self._init_tables()

    def _init_tables(self):
        self.conn.executescript("""
            CREATE VIRTUAL TABLE IF NOT EXISTS chunk_embeddings USING vec0(
                chunk_id TEXT PRIMARY KEY,
                embedding FLOAT[384]
            );
            CREATE VIRTUAL TABLE IF NOT EXISTS node_embeddings USING vec0(
                node_id TEXT PRIMARY KEY,
                embedding FLOAT[384]
            );
        """)
        self.conn.commit()

    def upsert_chunk(self, chunk_id, doc_id, embedding, text, metadata):
        vec_bytes = self._serialize(embedding)
        self.conn.execute(
            "INSERT OR REPLACE INTO chunk_embeddings(chunk_id, embedding) VALUES (?, ?)",
            (chunk_id, vec_bytes)
        )
        self.conn.commit()

    def upsert_node(self, node_id, embedding, label, node_type):
        vec_bytes = self._serialize(embedding)
        self.conn.execute(
            "INSERT OR REPLACE INTO node_embeddings(node_id, embedding) VALUES (?, ?)",
            (node_id, vec_bytes)
        )
        self.conn.commit()

    def query_chunks(self, embedding, top_k, filter_doc_ids=None) -> List[Tuple[str, str, float]]:
        vec_bytes = self._serialize(embedding)
        rows = self.conn.execute("""
            SELECT chunk_id, distance
            FROM chunk_embeddings
            WHERE embedding MATCH ?
            ORDER BY distance
            LIMIT ?
        """, (vec_bytes, top_k)).fetchall()
        # distance from sqlite-vec is L2; since vectors are normalized, L2 ≈ 2*(1-cosine)
        return [(row[0], None, 1 - (row[1] ** 2 / 2)) for row in rows]

    def query_nodes(self, embedding, top_k, filter_type=None) -> List[Tuple[str, float]]:
        vec_bytes = self._serialize(embedding)
        rows = self.conn.execute("""
            SELECT node_id, distance
            FROM node_embeddings
            WHERE embedding MATCH ?
            ORDER BY distance
            LIMIT ?
        """, (vec_bytes, top_k * 3)).fetchall()
        return [(row[0], 1 - (row[1] ** 2 / 2)) for row in rows[:top_k]]

    def delete_by_doc(self, doc_id):
        # Requires joining with chunk metadata table
        chunk_ids = self.conn.execute(
            "SELECT chunk_id FROM chunks WHERE doc_id = ?", (doc_id,)
        ).fetchall()
        for (cid,) in chunk_ids:
            self.conn.execute("DELETE FROM chunk_embeddings WHERE chunk_id = ?", (cid,))
        self.conn.commit()

    def _serialize(self, embedding: np.ndarray) -> bytes:
        return struct.pack(f"{self.EMBEDDING_DIM}f", *embedding.astype(np.float32))

4.6 Insight Engine

Purpose

The insight engine is what makes ChronoMap more than a graph tool. It actively mines the graph for patterns that the user would never find manually:

    Blind Spots: Pairs of nodes that are semantically close (similar embeddings) but graphically distant (long shortest path or in different communities). These are ideas you know well but have never explicitly connected.
    Knowledge Islands: Disconnected components or communities that share no edges with the rest of the graph.
    Obsession Tracker: Concepts whose mention count per time window is unusually high relative to historical baseline.
    Forgotten Ideas: Nodes that had high activity in the past but zero mentions in recent months.

Python

# insight/blind_spot_detector.py

import networkx as nx
import numpy as np
from typing import List
from dataclasses import dataclass

@dataclass
class BlindSpot:
    node_a_id: str
    node_a_label: str
    node_b_id: str
    node_b_label: str
    semantic_similarity: float   # 0.0-1.0
    graph_distance: int          # Shortest path length (or -1 if disconnected)
    score: float                 # Combined blind-spot score
    bridge_concepts: List[str]   # Concepts that might bridge A and B

class BlindSpotDetector:
    """
    Scalable blind spot detection using kNN from vector index
    rather than all-pairs O(n²) loop.
    
    Algorithm:
    1. For each node N, retrieve top-k semantic neighbors via vector index
    2. For each neighbor M of N:
       - Compute graph distance between N and M
       - If similarity > THRESHOLD and distance > DISTANCE_THRESHOLD:
         → Flag as blind spot candidate
    3. Score = semantic_similarity × log(graph_distance + 1) × recency_boost
    4. Deduplicate (A,B) and (B,A) pairs
    5. Return top-N blind spots ranked by score
    """
    SIM_THRESHOLD      = 0.75
    DISTANCE_THRESHOLD = 3       # Must be > 3 hops apart (or disconnected)
    KNN_K              = 20      # Semantic neighbors to check per node
    MAX_RESULTS        = 50

    def __init__(self, repo, vector_store):
        self.repo = repo
        self.vector_store = vector_store

    def detect(self, graph: nx.Graph) -> List[BlindSpot]:
        all_nodes = list(graph.nodes(data=True))
        candidates = {}  # (a_id, b_id) → BlindSpot

        for node_id, node_data in all_nodes:
            node = self.repo.get_node(node_id)
            if not node or not node.embedding:
                continue

            # Get top-k semantic neighbors from vector index
            neighbors = self.vector_store.query_nodes(
                embedding=np.array(node.embedding),
                top_k=self.KNN_K,
            )

            for neighbor_id, similarity in neighbors:
                if similarity < self.SIM_THRESHOLD:
                    continue
                if neighbor_id == node_id:
                    continue
                if not graph.has_node(neighbor_id):
                    continue

                # Compute graph distance
                try:
                    path_length = nx.shortest_path_length(graph, node_id, neighbor_id)
                except nx.NetworkXNoPath:
                    path_length = 999  # Disconnected components

                if path_length <= self.DISTANCE_THRESHOLD:
                    continue  # Already well-connected — not a blind spot

                pair_key = tuple(sorted([node_id, neighbor_id]))
                if pair_key in candidates:
                    continue

                score = similarity * np.log1p(path_length)
                neighbor_node = self.repo.get_node(neighbor_id)

                candidates[pair_key] = BlindSpot(
                    node_a_id=node_id,
                    node_a_label=node_data.get('label', node_id),
                    node_b_id=neighbor_id,
                    node_b_label=neighbor_node.label if neighbor_node else neighbor_id,
                    semantic_similarity=float(similarity),
                    graph_distance=path_length if path_length < 999 else -1,
                    score=float(score),
                    bridge_concepts=self._find_bridge_concepts(node_id, neighbor_id, graph),
                )

        results = sorted(candidates.values(), key=lambda x: x.score, reverse=True)
        return results[:self.MAX_RESULTS]

    def _find_bridge_concepts(self, a_id: str, b_id: str, graph: nx.Graph) -> List[str]:
        """Find concept nodes that are neighbors of both A and B — potential bridges."""
        if not graph.has_node(a_id) or not graph.has_node(b_id):
            return []
        a_neighbors = set(graph.neighbors(a_id))
        b_neighbors = set(graph.neighbors(b_id))
        shared = a_neighbors & b_neighbors
        return [graph.nodes[n].get('label', n) for n in list(shared)[:3]]

Python

# insight/island_detector.py

import networkx as nx
from dataclasses import dataclass
from typing import List

@dataclass
class KnowledgeIsland:
    island_id: str
    node_ids: List[str]
    node_labels: List[str]
    node_count: int
    dominant_topic: str
    isolation_score: float   # 0.0-1.0: how isolated vs. rest of graph

class IslandDetector:
    MIN_ISLAND_SIZE = 3    # Ignore trivially small components
    MAX_ISLAND_SIZE = 200  # Don't flag the main component

    def __init__(self, repo):
        self.repo = repo

    def detect(self, graph: nx.Graph) -> List[KnowledgeIsland]:
        components = list(nx.connected_components(graph))
        total_nodes = graph.number_of_nodes()
        islands = []

        for i, component in enumerate(components):
            if len(component) < self.MIN_ISLAND_SIZE:
                continue
            if len(component) > self.MAX_ISLAND_SIZE:
                continue  # This is likely the main cluster

            subgraph = graph.subgraph(component)
            node_ids = list(component)
            nodes = [self.repo.get_node(nid) for nid in node_ids if self.repo.get_node(nid)]
            labels = [n.label for n in nodes if n]

            # Dominant topic: most common entity_label or topic in this island
            from collections import Counter
            topic_counts = Counter(n.entity_label for n in nodes if n)
            dominant_topic = topic_counts.most_common(1)[0][0] if topic_counts else "Unknown"

            # Isolation score: island size as fraction of total (smaller = more isolated)
            isolation_score = 1.0 - (len(component) / total_nodes)

            islands.append(KnowledgeIsland(
                island_id=f"island_{i}",
                node_ids=node_ids,
                node_labels=labels[:10],
                node_count=len(component),
                dominant_topic=dominant_topic,
                isolation_score=isolation_score,
            ))

        return sorted(islands, key=lambda x: x.isolation_score, reverse=True)

Python

# insight/obsession_tracker.py

from datetime import datetime, timedelta
from dataclasses import dataclass
from typing import List
import statistics

@dataclass
class ObsessionPeriod:
    node_id: str
    node_label: str
    period_start: datetime
    period_end: datetime
    mention_count: int
    z_score: float            # Standard deviations above historical mean
    is_current: bool          # Is this period still active?

class ObsessionTracker:
    """
    Detects concepts with unusually high mention density in a given
    time window relative to historical baseline.
    
    A concept is an "obsession" if its mention count in a window
    is >= 2 standard deviations above its own historical mean.
    """
    WINDOW_DAYS      = 30   # Size of each time window
    MIN_MENTIONS     = 3    # Minimum mentions to qualify
    Z_SCORE_THRESHOLD = 2.0

    def __init__(self, repo):
        self.repo = repo

    def detect(self, top_n: int = 20) -> List[ObsessionPeriod]:
        now = datetime.utcnow()
        obsessions = []

        # Get all nodes with temporal mention series
        nodes = self.repo.get_nodes_with_mention_series()
        
        for node_id, label, mention_series in nodes:
            # mention_series: list of (window_start, mention_count) tuples
            if len(mention_series) < 3:
                continue

            counts = [count for _, count in mention_series]
            mean = statistics.mean(counts)
            stdev = statistics.stdev(counts) if len(counts) > 1 else 0

            for window_start, count in mention_series:
                window_end = window_start + timedelta(days=self.WINDOW_DAYS)
                if count < self.MIN_MENTIONS:
                    continue
                z = (count - mean) / stdev if stdev > 0 else 0.0
                if z >= self.Z_SCORE_THRESHOLD:
                    is_current = (now - window_end).days <= self.WINDOW_DAYS
                    obsessions.append(ObsessionPeriod(
                        node_id=node_id,
                        node_label=label,
                        period_start=window_start,
                        period_end=window_end,
                        mention_count=count,
                        z_score=z,
                        is_current=is_current,
                    ))

        return sorted(obsessions, key=lambda x: x.z_score, reverse=True)[:top_n]

Python

# insight/forgotten_idea_detector.py

from datetime import datetime, timedelta
from dataclasses import dataclass
from typing import List

@dataclass
class ForgottenIdea:
    node_id: str
    node_label: str
    peak_mention_count: int   # Max mentions in any 30-day window
    peak_period_start: datetime
    days_since_last_mention: int
    forget_score: float       # peak × recency_gap

class ForgottenIdeaDetector:
    DORMANCY_DAYS = 90        # Not seen in 3 months = candidate
    MIN_PEAK_MENTIONS = 5     # Must have been significant at some point

    def __init__(self, repo):
        self.repo = repo

    def detect(self, top_n: int = 20) -> List[ForgottenIdea]:
        now = datetime.utcnow()
        cutoff = now - timedelta(days=self.DORMANCY_DAYS)
        forgotten = []

        nodes = self.repo.get_dormant_nodes(last_seen_before=cutoff)
        for node in nodes:
            peak = self.repo.get_peak_mention_window(node.node_id)
            if not peak or peak['count'] < self.MIN_PEAK_MENTIONS:
                continue

            days_dormant = (now - node.last_seen).days
            forget_score = peak['count'] * (days_dormant / 365)

            forgotten.append(ForgottenIdea(
                node_id=node.node_id,
                node_label=node.label,
                peak_mention_count=peak['count'],
                peak_period_start=peak['window_start'],
                days_since_last_mention=days_dormant,
                forget_score=forget_score,
            ))

        return sorted(forgotten, key=lambda x: x.forget_score, reverse=True)[:top_n]

4.7 Watch Daemon

Purpose

The watch daemon monitors configured source directories and schedules re-ingestion whenever files are created, modified, or deleted.

Python

# watch_daemon.py

import time
import logging
from pathlib import Path
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler, FileCreatedEvent, FileModifiedEvent, FileDeletedEvent
from queue import Queue, Empty
from threading import Thread

logger = logging.getLogger("chronomap.daemon")

SUPPORTED_EXTENSIONS = {
    '.md', '.markdown', '.mdx',
    '.pdf',
    '.txt',
    '.py', '.js', '.ts', '.go', '.rs', '.java', '.cpp',
    '.ics', '.ical',
    '.mbox',
    '.csv',  # Kindle/Kobo highlight exports
}

class ChronoMapEventHandler(FileSystemEventHandler):
    def __init__(self, queue: Queue, debounce_seconds: float = 2.0):
        self.queue = queue
        self.debounce_seconds = debounce_seconds
        self._pending = {}  # path → scheduled time

    def on_created(self, event):
        if not event.is_directory:
            self._schedule(event.src_path, 'created')

    def on_modified(self, event):
        if not event.is_directory:
            self._schedule(event.src_path, 'modified')

    def on_deleted(self, event):
        if not event.is_directory:
            self.queue.put({'type': 'deleted', 'path': event.src_path})

    def _schedule(self, path: str, event_type: str):
        ext = Path(path).suffix.lower()
        if ext not in SUPPORTED_EXTENSIONS:
            return
        # Debounce: only queue after file has been stable for N seconds
        self._pending[path] = (time.time() + self.debounce_seconds, event_type)

    def flush_pending(self):
        """Called periodically to drain debounced events."""
        now = time.time()
        to_remove = []
        for path, (scheduled_at, event_type) in self._pending.items():
            if now >= scheduled_at:
                self.queue.put({'type': event_type, 'path': path})
                to_remove.append(path)
        for path in to_remove:
            del self._pending[path]

class WatchDaemon:
    def __init__(self, watch_paths: list[str], ingestion_pipeline, config):
        self.watch_paths = watch_paths
        self.pipeline = ingestion_pipeline
        self.config = config
        self.queue = Queue(maxsize=10_000)
        self.observer = Observer()
        self.handler = ChronoMapEventHandler(self.queue)
        self._running = False

    def start(self):
        self._running = True
        for path in self.watch_paths:
            self.observer.schedule(self.handler, path, recursive=True)
        self.observer.start()
        
        # Initial full scan at startup
        Thread(target=self._initial_scan, daemon=True).start()
        
        # Debounce flusher
        Thread(target=self._flush_loop, daemon=True).start()
        
        # Queue consumer
        Thread(target=self._consume_loop, daemon=True).start()
        
        logger.info(f"WatchDaemon started. Monitoring: {self.watch_paths}")

    def stop(self):
        self._running = False
        self.observer.stop()
        self.observer.join()

    def _initial_scan(self):
        """Scan all watched directories on startup for new/changed files."""
        for watch_path in self.watch_paths:
            for ext in SUPPORTED_EXTENSIONS:
                for file in Path(watch_path).rglob(f'*{ext}'):
                    self.queue.put({'type': 'initial_scan', 'path': str(file)})

    def _flush_loop(self):
        while self._running:
            self.handler.flush_pending()
            time.sleep(0.5)

    def _consume_loop(self):
        while self._running:
            try:
                event = self.queue.get(timeout=1.0)
                self._handle_event(event)
                self.queue.task_done()
            except Empty:
                continue
            except Exception as e:
                logger.error(f"Error handling event: {e}", exc_info=True)

    def _handle_event(self, event: dict):
        path = event['path']
        event_type = event['type']

        if event_type == 'deleted':
            logger.info(f"File deleted: {path} — removing from graph")
            self.pipeline.handle_deletion(path)
            return

        # Check if file changed since last ingest (checksum comparison)
        if event_type != 'initial_scan':
            if not self.pipeline.has_changed(path):
                return

        logger.info(f"Ingesting [{event_type}]: {path}")
        try:
            self.pipeline.run(path)
        except Exception as e:
            logger.error(f"Ingestion failed for {path}: {e}", exc_info=True)

4.8 REST API Server

Purpose

FastAPI server serving the Svelte UI and all API endpoints. Provides graph data, search, temporal queries, insights, and WebSocket updates.

Python

# api/server.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Query, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles
from typing import Optional, List
from datetime import datetime
import asyncio
import json

app = FastAPI(title="ChronoMap API", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173", "http://localhost:5174"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# WebSocket connection manager for real-time graph updates
class ConnectionManager:
    def __init__(self):
        self.active: List[WebSocket] = []

    async def connect(self, ws: WebSocket):
        await ws.accept()
        self.active.append(ws)

    def disconnect(self, ws: WebSocket):
        self.active.remove(ws)

    async def broadcast(self, message: dict):
        for ws in self.active:
            try:
                await ws.send_json(message)
            except Exception:
                pass

manager = ConnectionManager()

@app.websocket("/ws/graph-updates")
async def websocket_updates(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            await websocket.receive_text()  # Keep alive
    except WebSocketDisconnect:
        manager.disconnect(websocket)

4.9 Graph UI (Svelte + Three.js + D3)

Purpose

The Svelte frontend provides the primary user interface: an interactive 3D/2D knowledge graph with a time slider, semantic search, node detail panels, and insight dashboards.

Component Overview

text

ui/src/
├── App.svelte                     # Root component, routing
├── pages/
│   ├── GraphView.svelte           # Main graph canvas page
│   ├── Search.svelte              # Semantic search results
│   ├── Insights.svelte            # Blind spots, islands, obsessions
│   └── Settings.svelte            # Config, watched paths, ingestors
├── components/
│   ├── GraphCanvas.svelte         # Three.js + D3 force graph renderer
│   ├── TimeSlider.svelte          # Temporal slider with keypoints
│   ├── NodeDetail.svelte          # Side panel: node info + provenance
│   ├── BlindSpotPanel.svelte      # Blind spot list + highlight on graph
│   ├── SearchBar.svelte           # Semantic search input + results
│   ├── ClusterView.svelte         # Topic-level zoom-out view
│   └── ObsessionTracker.svelte    # Timeline bar chart of obsessions
├── stores/
│   └── graph.ts                   # Svelte writable stores for graph state
├── lib/
│   ├── graphRenderer.ts           # Three.js scene setup and rendering
│   ├── forceSimulation.ts         # D3 force layout engine
│   ├── lod.ts                     # Level-of-detail clustering logic
│   └── api.ts                     # API client
└── types/
    └── index.ts                   # TypeScript interfaces

GraphCanvas Component

svelte

<!-- components/GraphCanvas.svelte -->
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
  import { forceSimulation, forceLink, forceManyBody, forceCenter, forceCollide } from 'd3-force';
  import type { GraphData, KGNode, KGEdge } from '../types';
  import { selectedNode, graphData, renderMode } from '../stores/graph';

  export let data: GraphData;

  let container: HTMLDivElement;
  let scene: THREE.Scene;
  let camera: THREE.PerspectiveCamera;
  let renderer: THREE.WebGLRenderer;
  let controls: OrbitControls;
  let nodeMeshes: Map<string, THREE.Mesh> = new Map();
  let edgeLines: Map<string, THREE.Line> = new Map();
  let animationId: number;

  const NODE_COLORS = {
    entity: { PERSON: 0x4f9eff, ORG: 0xff6b4f, GPE: 0x4fff9e, CONCEPT: 0xffd700 },
    concept: 0xffd700,
    topic: 0xff4fff,
    document: 0xaaaaaa,
  };

  onMount(() => {
    initScene();
    buildGraph(data);
    animate();
  });

  onDestroy(() => {
    cancelAnimationFrame(animationId);
    renderer.dispose();
  });

  function initScene() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a0a1a);
    scene.fog = new THREE.Fog(0x0a0a1a, 200, 600);

    camera = new THREE.PerspectiveCamera(75, container.clientWidth / container.clientHeight, 0.1, 1000);
    camera.position.set(0, 0, 200);

    renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
    renderer.setSize(container.clientWidth, container.clientHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    container.appendChild(renderer.domElement);

    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;

    // Ambient + point lights for glow effect
    scene.add(new THREE.AmbientLight(0x112244, 0.8));
    const pointLight = new THREE.PointLight(0x4488ff, 1.5, 300);
    scene.add(pointLight);
  }

  function buildGraph(graphData: GraphData) {
    // Run D3 force simulation to compute positions
    const simulation = forceSimulation(graphData.nodes)
      .force('link', forceLink(graphData.edges).id((d: any) => d.node_id).distance(30).strength(0.3))
      .force('charge', forceManyBody().strength(-80))
      .force('center', forceCenter(0, 0))
      .force('collide', forceCollide(8))
      .stop();

    // Run synchronously for initial layout
    for (let i = 0; i < 300; i++) simulation.tick();

    // Build Three.js meshes for nodes
    graphData.nodes.forEach(node => {
      const geometry = new THREE.SphereGeometry(getNodeRadius(node), 16, 16);
      const material = new THREE.MeshPhongMaterial({
        color: getNodeColor(node),
        emissive: getNodeColor(node),
        emissiveIntensity: 0.3,
        transparent: true,
        opacity: 0.9,
      });
      const mesh = new THREE.Mesh(geometry, material);
      mesh.position.set((node as any).x || 0, (node as any).y || 0, Math.random() * 20 - 10);
      mesh.userData = { node_id: node.node_id, label: node.label };
      scene.add(mesh);
      nodeMeshes.set(node.node_id, mesh);
    });

    // Build lines for edges
    graphData.edges.forEach(edge => {
      const srcMesh = nodeMeshes.get(edge.source_node_id);
      const tgtMesh = nodeMeshes.get(edge.target_node_id);
      if (!srcMesh || !tgtMesh) return;

      const points = [srcMesh.position.clone(), tgtMesh.position.clone()];
      const geometry = new THREE.BufferGeometry().setFromPoints(points);
      const material = new THREE.LineBasicMaterial({
        color: 0x334466,
        transparent: true,
        opacity: Math.max(0.1, edge.weight * 0.6),
      });
      const line = new THREE.Line(geometry, material);
      scene.add(line);
      edgeLines.set(edge.edge_id, line);
    });
  }

  function animate() {
    animationId = requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
  }

  function getNodeRadius(node: KGNode): number {
    return Math.max(2, Math.min(10, 2 + Math.log1p(node.mention_count)));
  }

  function getNodeColor(node: KGNode): number {
    if (node.node_type === 'entity') {
      return NODE_COLORS.entity[node.entity_label as keyof typeof NODE_COLORS.entity] || 0x88aaff;
    }
    return NODE_COLORS[node.node_type as keyof typeof NODE_COLORS] as number || 0x888888;
  }

  // Update graph positions when time slider changes
  $: if (data) {
    buildGraph(data);
  }
</script>

<div bind:this={container} class="w-full h-full" />

Time Slider Component

svelte

<!-- components/TimeSlider.svelte -->
<script lang="ts">
  import { createEventDispatcher, onMount } from 'svelte';
  import type { TimelineKeypoint } from '../types';

  export let keypoints: TimelineKeypoint[] = [];
  export let value: string;  // ISO datetime string

  const dispatch = createEventDispatcher();

  let min: number;
  let max: number;
  let current: number;

  onMount(() => {
    if (keypoints.length) {
      min = new Date(keypoints[0].timestamp).getTime();
      max = new Date(keypoints[keypoints.length - 1].timestamp).getTime();
      current = max;  // Default: show current graph
    }
  });

  function handleChange(event: Event) {
    const ts = parseInt((event.target as HTMLInputElement).value);
    current = ts;
    dispatch('change', { timestamp: new Date(ts).toISOString() });
  }

  function formatDate(ts: number): string {
    return new Date(ts).toLocaleDateString('en-US', { year: 'numeric', month: 'short' });
  }
</script>

<div class="chronomap-timeslider">
  <div class="flex items-center gap-4">
    <span class="text-xs text-slate-400">{formatDate(min)}</span>
    <div class="flex-1 relative">
      <input
        type="range"
        {min} {max} bind:value={current}
        on:input={handleChange}
        class="w-full accent-blue-500"
      />
      <!-- Keypoint markers -->
      {#each keypoints as kp}
        <div
          class="absolute top-0 w-1 h-3 bg-blue-400 opacity-50"
          style="left: {((new Date(kp.timestamp).getTime() - min) / (max - min)) * 100}%"
          title={kp.label}
        />
      {/each}
    </div>
    <span class="text-xs text-slate-400">{formatDate(max)}</span>
  </div>
  <div class="text-center text-sm text-blue-300 mt-1">
    Viewing: {formatDate(current)}
  </div>
</div>

5. Data Models & Schemas
5.1 SQLite Schema

SQL

-- migrations/001_initial.sql

-- Source documents
CREATE TABLE IF NOT EXISTS documents (
    doc_id          TEXT PRIMARY KEY,
    source_type     TEXT NOT NULL,        -- markdown|pdf|browser|email|code|highlight|calendar
    uri             TEXT NOT NULL UNIQUE, -- Absolute path or virtual URI
    title           TEXT,
    author          TEXT,
    created_at      DATETIME,
    modified_at     DATETIME,
    ingested_at     DATETIME NOT NULL,
    checksum        TEXT NOT NULL,        -- SHA256 of full_text (change detection)
    char_count      INTEGER,
    source_metadata TEXT,                 -- JSON blob of source-specific fields
    last_ingest_at  DATETIME             -- Updated on every re-ingest
);

-- Text chunks (sliding window segments of documents)
CREATE TABLE IF NOT EXISTS chunks (
    chunk_id        TEXT PRIMARY KEY,
    doc_id          TEXT NOT NULL REFERENCES documents(doc_id) ON DELETE CASCADE,
    text            TEXT NOT NULL,
    token_start     INTEGER,
    token_end       INTEGER,
    char_start      INTEGER,
    char_end        INTEGER,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Knowledge graph nodes
CREATE TABLE IF NOT EXISTS nodes (
    node_id         TEXT PRIMARY KEY,
    label           TEXT NOT NULL,        -- Canonical surface form
    normalized_label TEXT NOT NULL,       -- Lowercased, whitespace-normalized
    node_type       TEXT NOT NULL,        -- entity|concept|topic|document
    entity_label    TEXT,                 -- PERSON|ORG|GPE|CONCEPT|TOPIC etc.
    first_seen      DATETIME NOT NULL,
    last_seen       DATETIME NOT NULL,
    mention_count   INTEGER NOT NULL DEFAULT 1,
    doc_ids         TEXT NOT NULL,        -- JSON array of doc_id strings
    embedding       BLOB,                 -- 384-dim float32 vector (binary)
    is_merged       BOOLEAN DEFAULT FALSE,
    merged_into     TEXT REFERENCES nodes(node_id),
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Knowledge graph edges
CREATE TABLE IF NOT EXISTS edges (
    edge_id         TEXT PRIMARY KEY,
    source_node_id  TEXT NOT NULL REFERENCES nodes(node_id),
    target_node_id  TEXT NOT NULL REFERENCES nodes(node_id),
    edge_type       TEXT NOT NULL,        -- cooccurrence|relation|semantic|wikilink
    weight          REAL NOT NULL DEFAULT 0.1,
    predicate       TEXT,                 -- Verb lemma if from relation extraction
    first_seen      DATETIME NOT NULL,
    last_seen       DATETIME NOT NULL,
    doc_ids         TEXT NOT NULL,        -- JSON array
    confidence      REAL NOT NULL DEFAULT 0.7,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Entity mentions: which chunks contain which nodes
CREATE TABLE IF NOT EXISTS mentions (
    mention_id      TEXT PRIMARY KEY,
    node_id         TEXT NOT NULL REFERENCES nodes(node_id),
    doc_id          TEXT NOT NULL REFERENCES documents(doc_id),
    chunk_id        TEXT NOT NULL REFERENCES chunks(chunk_id),
    mention_text    TEXT NOT NULL,        -- Exact surface form found in text
    start_char      INTEGER,
    end_char        INTEGER,
    confidence      REAL DEFAULT 1.0,
    extraction_method TEXT,               -- ner|keybert|relation|manual
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Topic assignments per document
CREATE TABLE IF NOT EXISTS topics (
    topic_id        TEXT PRIMARY KEY,
    topic_label     TEXT NOT NULL,        -- Human-readable: "machine learning / neural networks"
    bertopic_id     INTEGER,              -- BERTopic internal ID
    top_words       TEXT,                 -- JSON array of (word, score) pairs
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS document_topics (
    doc_id          TEXT NOT NULL REFERENCES documents(doc_id),
    topic_id        TEXT NOT NULL REFERENCES topics(topic_id),
    probability     REAL DEFAULT 1.0,
    PRIMARY KEY (doc_id, topic_id)
);

-- Node aliases (from merge events)
CREATE TABLE IF NOT EXISTS node_aliases (
    alias_label     TEXT NOT NULL,
    canonical_node_id TEXT NOT NULL REFERENCES nodes(node_id),
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (alias_label, canonical_node_id)
);

-- Temporal mention series (pre-aggregated for obsession tracking)
CREATE TABLE IF NOT EXISTS mention_timeseries (
    node_id         TEXT NOT NULL REFERENCES nodes(node_id),
    window_start    DATETIME NOT NULL,    -- Start of 30-day window
    mention_count   INTEGER NOT NULL DEFAULT 0,
    doc_count       INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (node_id, window_start)
);

-- Cached insight results
CREATE TABLE IF NOT EXISTS insights (
    insight_id      TEXT PRIMARY KEY,
    insight_type    TEXT NOT NULL,        -- blind_spot|island|obsession|forgotten
    computed_at     DATETIME NOT NULL,
    payload         TEXT NOT NULL,        -- JSON blob of the insight struct
    is_dismissed    BOOLEAN DEFAULT FALSE
);

-- Settings
CREATE TABLE IF NOT EXISTS settings (
    key             TEXT PRIMARY KEY,
    value           TEXT NOT NULL,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Ingestor configuration and state
CREATE TABLE IF NOT EXISTS ingestor_state (
    source_type     TEXT PRIMARY KEY,
    enabled         BOOLEAN NOT NULL DEFAULT TRUE,
    watch_paths     TEXT,                 -- JSON array of paths
    last_full_scan  DATETIME,
    config          TEXT                  -- JSON blob
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_nodes_normalized_label   ON nodes(normalized_label);
CREATE INDEX IF NOT EXISTS idx_nodes_node_type          ON nodes(node_type);
CREATE INDEX IF NOT EXISTS idx_nodes_first_seen         ON nodes(first_seen);
CREATE INDEX IF NOT EXISTS idx_nodes_last_seen          ON nodes(last_seen);
CREATE INDEX IF NOT EXISTS idx_edges_source             ON edges(source_node_id);
CREATE INDEX IF NOT EXISTS idx_edges_target             ON edges(target_node_id);
CREATE INDEX IF NOT EXISTS idx_edges_first_seen         ON edges(first_seen);
CREATE INDEX IF NOT EXISTS idx_mentions_node            ON mentions(node_id);
CREATE INDEX IF NOT EXISTS idx_mentions_doc             ON mentions(doc_id);
CREATE INDEX IF NOT EXISTS idx_chunks_doc               ON chunks(doc_id);
CREATE INDEX IF NOT EXISTS idx_documents_source_type    ON documents(source_type);
CREATE INDEX IF NOT EXISTS idx_documents_modified       ON documents(modified_at);
CREATE INDEX IF NOT EXISTS idx_timeseries_node          ON mention_timeseries(node_id);

6. API Specifications
6.1 Graph Endpoints

text

GET  /api/v1/graph
     Query: ?limit_nodes=500&min_weight=0.05&node_type=&topic_id=
     Returns: { nodes: KGNode[], edges: KGEdge[], stats: GraphStats }

GET  /api/v1/graph/subgraph/:node_id
     Query: ?depth=2
     Returns: { nodes: KGNode[], edges: KGEdge[] }
     (Returns N-hop neighborhood of a given node)

GET  /api/v1/nodes/:node_id
     Returns: KGNode (with provenance: source documents + chunk snippets)

GET  /api/v1/nodes/:node_id/neighbors
     Query: ?depth=1&edge_type=
     Returns: { nodes: KGNode[], edges: KGEdge[] }

GET  /api/v1/stats
     Returns: {
       total_nodes, total_edges, total_documents,
       total_chunks, top_entities[], top_concepts[],
       source_type_breakdown{}, last_ingest_at
     }

6.2 Temporal Endpoints

text

GET  /api/v1/temporal/graph
     Query: ?timestamp=2024-06-01T00:00:00Z
     Returns: { nodes: KGNode[], edges: KGEdge[], timestamp: str }

GET  /api/v1/temporal/diff
     Query: ?t1=2024-01-01T00:00:00Z&t2=2024-06-01T00:00:00Z
     Returns: { nodes_added[], nodes_removed[], edges_added[], edges_removed[], summary }

GET  /api/v1/temporal/keypoints
     Returns: [{ timestamp, label, node_count, edge_count, event_type }]

GET  /api/v1/temporal/node/:node_id
     Returns: { node_id, label, appearances: [{ timestamp, mention_count, doc_count }] }

6.3 Search Endpoints

text

POST /api/v1/search
     Body: { query: str, top_k: int = 10, search_type: "semantic"|"keyword"|"hybrid" }
     Returns: {
       results: [{
         chunk_id, doc_id, doc_title, source_type,
         text_snippet, score, node_matches: KGNode[]
       }]
     }

GET  /api/v1/search/nodes
     Query: ?q=&node_type=&limit=20
     Returns: KGNode[]

POST /api/v1/search/similar_nodes
     Body: { node_id: str, top_k: int = 10 }
     Returns: [{ node: KGNode, similarity: float }]

6.4 Insight Endpoints

text

GET  /api/v1/insights/blind-spots
     Query: ?limit=20&refresh=false
     Returns: [BlindSpot]

GET  /api/v1/insights/islands
     Query: ?limit=10
     Returns: [KnowledgeIsland]

GET  /api/v1/insights/obsessions
     Query: ?limit=20&current_only=false
     Returns: [ObsessionPeriod]

GET  /api/v1/insights/forgotten
     Query: ?limit=20
     Returns: [ForgottenIdea]

POST /api/v1/insights/:insight_id/dismiss
     Returns: { success: bool }

6.5 Document & Ingestor Endpoints

text

GET  /api/v1/documents
     Query: ?source_type=&limit=50&offset=0
     Returns: { total, items: Document[] }

GET  /api/v1/documents/:doc_id
     Returns: Document (with chunks, entities, topics)

POST /api/v1/ingest/trigger
     Body: { path: str }
     Returns: { job_id, status }

GET  /api/v1/ingest/status
     Returns: { queue_depth, active_job, last_completed, errors[] }

GET  /api/v1/settings
     Returns: Settings

PUT  /api/v1/settings
     Body: Partial<Settings>
     Returns: Settings

6.6 Real-Time WebSocket

text

WS   /ws/graph-updates

Events pushed by server:
  { type: "node_added",    payload: KGNode }
  { type: "node_updated",  payload: KGNode }
  { type: "edge_added",    payload: KGEdge }
  { type: "ingest_started", payload: { doc_id, uri, source_type } }
  { type: "ingest_complete", payload: { doc_id, nodes_added, edges_added } }
  { type: "insight_ready",  payload: { insight_type } }

7. Directory Structure

text

chronomap/
├── cmd/
│   └── chronomap/
│       └── main.py                      # Entry point: start API + watch daemon
│
├── ingestors/
│   ├── __init__.py
│   ├── base.py                          # BaseIngestor, Document, TextBlock
│   ├── registry.py                      # IngestorRegistry
│   ├── markdown_ingestor.py             # .md/.mdx files + frontmatter + WikiLinks
│   ├── pdf_ingestor.py                  # PDFs: text + highlights (PyMuPDF)
│   ├── browser_ingestor.py              # Chrome/Firefox history + bookmarks
│   ├── email_ingestor.py                # mbox/maildir archives
│   ├── code_ingestor.py                 # Source code files (all major languages)
│   ├── highlight_ingestor.py            # Kindle/Kobo CSV exports, Readwise
│   └── calendar_ingestor.py             # iCal/ics files
│
├── extractor/
│   ├── __init__.py
│   ├── chunker.py                       # Sliding-window sentence-aware chunker
│   ├── entity_extractor.py              # spaCy NER
│   ├── concept_extractor.py             # KeyBERT keyphrases
│   ├── embedder.py                      # SentenceTransformers (all-MiniLM-L6-v2)
│   ├── topic_modeler.py                 # BERTopic
│   ├── relation_extractor.py            # Dependency parse → triples
│   └── pipeline.py                      # Orchestrates all extractors in sequence
│
├── graph/
│   ├── __init__.py
│   ├── types.py                         # KGNode, KGEdge dataclasses
│   ├── builder.py                       # GraphBuilder: entity/edge upsert logic
│   ├── merger.py                        # EntityMerger: deduplication
│   ├── scorer.py                        # Edge weight recalculation
│   ├── temporal.py                      # TemporalGraph: graph_at(t), diff
│   └── community.py                     # Community detection (Louvain)
│
├── vector_store/
│   ├── __init__.py
│   ├── store.py                         # VectorStore ABC
│   ├── sqlite_vec_store.py              # sqlite-vec backend
│   └── chroma_store.py                  # Chroma backend (alternative)
│
├── insight/
│   ├── __init__.py
│   ├── blind_spot_detector.py           # BlindSpotDetector
│   ├── island_detector.py               # IslandDetector
│   ├── obsession_tracker.py             # ObsessionTracker
│   └── forgotten_idea_detector.py       # ForgottenIdeaDetector
│
├── storage/
│   ├── __init__.py
│   ├── db.py                            # SQLite connection + WAL config
│   ├── migrations.py                    # Migration runner
│   ├── migrations/
│   │   ├── 001_initial.sql
│   │   └── 002_timeseries.sql
│   ├── document_repo.py                 # CRUD for documents + chunks
│   ├── graph_repo.py                    # CRUD for nodes + edges
│   ├── mention_repo.py                  # CRUD for mentions
│   ├── topic_repo.py                    # CRUD for topics + assignments
│   ├── insight_repo.py                  # CRUD for cached insights
│   └── settings_repo.py                 # Key/value settings
│
├── api/
│   ├── __init__.py
│   ├── server.py                        # FastAPI app, CORS, static files
│   ├── routers/
│   │   ├── graph.py                     # /api/v1/graph endpoints
│   │   ├── temporal.py                  # /api/v1/temporal endpoints
│   │   ├── search.py                    # /api/v1/search endpoints
│   │   ├── insights.py                  # /api/v1/insights endpoints
│   │   ├── documents.py                 # /api/v1/documents endpoints
│   │   └── settings.py                  # /api/v1/settings endpoints
│   ├── websocket.py                     # WebSocket handler + ConnectionManager
│   └── dto.py                           # Pydantic request/response models
│
├── watch_daemon.py                      # WatchDaemon + ChronoMapEventHandler
├── ingestion_pipeline.py                # Orchestrates ingestor → extractor → graph
│
├── ui/                                  # Svelte frontend
│   ├── src/
│   │   ├── App.svelte
│   │   ├── pages/
│   │   │   ├── GraphView.svelte
│   │   │   ├── Search.svelte
│   │   │   ├── Insights.svelte
│   │   │   └── Settings.svelte
│   │   ├── components/
│   │   │   ├── GraphCanvas.svelte
│   │   │   ├── TimeSlider.svelte
│   │   │   ├── NodeDetail.svelte
│   │   │   ├── BlindSpotPanel.svelte
│   │   │   ├── SearchBar.svelte
│   │   │   ├── ClusterView.svelte
│   │   │   └── ObsessionTracker.svelte
│   │   ├── stores/
│   │   │   └── graph.ts
│   │   ├── lib/
│   │   │   ├── graphRenderer.ts
│   │   │   ├── forceSimulation.ts
│   │   │   ├── lod.ts                   # Level-of-detail clustering
│   │   │   └── api.ts
│   │   └── types/
│   │       └── index.ts
│   ├── package.json
│   ├── svelte.config.js
│   └── vite.config.ts
│
├── config/
│   ├── default_config.toml              # Default configuration
│   └── config_loader.py
│
├── tests/
│   ├── unit/
│   │   ├── test_chunker.py
│   │   ├── test_entity_extractor.py
│   │   ├── test_concept_extractor.py
│   │   ├── test_graph_builder.py
│   │   ├── test_merger.py
│   │   ├── test_temporal.py
│   │   ├── test_blind_spot_detector.py
│   │   └── test_transforms.py
│   ├── integration/
│   │   ├── test_ingestion_pipeline.py
│   │   ├── test_api_endpoints.py
│   │   └── test_vector_search.py
│   └── fixtures/
│       ├── sample.md
│       ├── sample.pdf
│       └── sample_history.db
│
├── data/                                # SQLite DB + blocklist cache (gitignored)
├── logs/                                # Log files (gitignored)
├── requirements.txt
├── pyproject.toml
├── Makefile
├── Dockerfile
├── docker-compose.yml
└── README.md

8. Configuration System
8.1 Config File (TOML)

Stored at ~/.chronomap/config.toml

toml

[general]
app_name = "ChronoMap"
data_dir = "~/.chronomap/data"
log_dir  = "~/.chronomap/logs"
log_level = "info"   # debug|info|warn|error

[api]
host = "127.0.0.1"
port = 8000
open_browser_on_start = true
cors_origins = ["http://localhost:5173"]

[ui]
theme = "dark"         # "dark" | "light"
default_render_mode = "3d"  # "3d" | "2d"
max_visible_nodes = 2000    # LOD threshold

[watch]
enabled = true
debounce_seconds = 2.0
paths = [
    "~/Documents",
    "~/Notes",
    "~/Downloads",
]
excluded_patterns = [
    "*.DS_Store",
    "*/.git/*",
    "*/node_modules/*",
    "*/.venv/*",
]

[ingestors.markdown]
enabled = true
extensions = [".md", ".markdown", ".mdx"]

[ingestors.pdf]
enabled = true
extract_highlights = true

[ingestors.browser]
enabled = true
browser = "chrome"    # "chrome" | "firefox" | "safari"
max_entries = 10000
min_visit_count = 1

[ingestors.email]
enabled = false       # Disabled by default (privacy-sensitive)
mbox_paths = []

[ingestors.code]
enabled = true
extensions = [".py", ".js", ".ts", ".go", ".rs", ".java", ".cpp", ".md"]
max_file_size_kb = 500
extract_comments_only = false

[ingestors.highlights]
enabled = true
formats = ["kindle_csv", "kobo_sqlite", "readwise_csv"]
paths = []

[ingestors.calendar]
enabled = true
ics_paths = []
include_google_calendar = false  # Requires OAuth (Phase 2)

[extraction]
spacy_model = "en_core_web_sm"   # "en_core_web_sm" | "en_core_web_trf"
embedding_model = "sentence-transformers/all-MiniLM-L6-v2"
chunk_size_tokens = 512
chunk_overlap_tokens = 64
entity_confidence_threshold = 0.7
concept_top_n = 10
min_concept_score = 0.15
enable_relation_extraction = true
enable_topic_modeling = true
topic_refit_interval_docs = 100   # Refit BERTopic every N new documents

[graph]
min_edge_weight = 0.05            # Edges below this are pruned in API responses
cooccurrence_weight_increment = 0.05
max_nodes_returned = 5000
enable_community_detection = true
community_algorithm = "louvain"   # "louvain" | "label_propagation"

[temporal]
staleness_days = 90               # Days after last_seen before edge/node fades
snapshot_granularity = "month"    # "day" | "week" | "month"

[vector_store]
backend = "sqlite_vec"            # "sqlite_vec" | "chroma"
embedding_dim = 384
chroma_path = "~/.chronomap/data/chroma"

[insights]
blind_spot_sim_threshold = 0.75
blind_spot_distance_threshold = 3
blind_spot_knn_k = 20
obsession_window_days = 30
obsession_z_score_threshold = 2.0
forgotten_dormancy_days = 90
forgotten_min_peak_mentions = 5
insight_cache_ttl_hours = 6       # How long to cache insight results

[storage]
db_path = "~/.chronomap/data/chronomap.db"
max_db_size_mb = 2000
log_retention_days = 90
vacuum_interval_days = 7

[logging]
level = "info"
file = "~/.chronomap/logs/chronomap.log"
max_size_mb = 100
max_backups = 5

8.2 Config Loader

Python

# config/config_loader.py

import tomllib
import os
from pathlib import Path
from dataclasses import dataclass

CONFIG_PATH = Path.home() / ".chronomap" / "config.toml"

def load_config(path: Path = CONFIG_PATH) -> dict:
    """Load config from TOML, creating default if missing."""
    if not path.exists():
        path.parent.mkdir(parents=True, exist_ok=True)
        default_path = Path(__file__).parent / "default_config.toml"
        import shutil
        shutil.copy(default_path, path)
        print(f"Created default config at {path}")

    with open(path, "rb") as f:
        config = tomllib.load(f)

    # Expand ~ in paths
    _expand_paths(config)
    return config

def _expand_paths(config: dict):
    """Recursively expand ~ in all string values that look like paths."""
    for key, value in config.items():
        if isinstance(value, dict):
            _expand_paths(value)
        elif isinstance(value, str) and value.startswith("~/"):
            config[key] = str(Path(value).expanduser())
        elif isinstance(value, list):
            config[key] = [
                str(Path(v).expanduser()) if isinstance(v, str) and v.startswith("~/") else v
                for v in value
            ]

9. Ingestor Deep Dive
9.1 Email Ingestor

Python

# ingestors/email_ingestor.py

import mailbox
import email
import hashlib
from pathlib import Path
from datetime import datetime
from email.utils import parsedate_to_datetime
from .base import BaseIngestor, Document, TextBlock

class EmailIngestor(BaseIngestor):
    """
    Ingests local email archives in mbox or maildir format.
    Disabled by default due to sensitivity of email content.
    Extracts: subject, sender, body text, received date.
    Does NOT ingest attachments (Phase 2).
    """

    def can_handle(self, uri: str) -> bool:
        p = Path(uri)
        return p.suffix.lower() == '.mbox' or (p.is_dir() and (p / 'cur').exists())

    def ingest(self, uri: str) -> Document:
        path = Path(uri)
        if path.suffix.lower() == '.mbox':
            mbox = mailbox.mbox(uri)
        else:
            mbox = mailbox.Maildir(uri)

        blocks = []
        full_text_parts = []
        senders = set()

        for message in mbox:
            subject = str(message.get('subject', ''))
            sender  = str(message.get('from', ''))
            date_str = message.get('date', '')
            body = self._get_body(message)

            try:
                msg_date = parsedate_to_datetime(date_str)
            except Exception:
                msg_date = None

            if not body.strip():
                continue

            block_text = f"Subject: {subject}\nFrom: {sender}\n\n{body}"
            blocks.append(TextBlock(
                text=block_text,
                block_type='paragraph',
                offset=len('\n'.join(full_text_parts)),
                metadata={'subject': subject, 'sender': sender, 'date': date_str}
            ))
            full_text_parts.append(block_text)
            if sender:
                senders.add(sender)

        full_text = '\n\n'.join(full_text_parts)
        return Document(
            doc_id=self._stable_id(uri),
            source_type='email',
            uri=str(path.resolve()),
            title=f"Email Archive: {path.name}",
            author=None,
            created_at=datetime.utcnow(),
            modified_at=datetime.fromtimestamp(path.stat().st_mtime),
            ingested_at=datetime.utcnow(),
            full_text=full_text,
            blocks=blocks,
            source_metadata={'sender_count': len(senders), 'message_count': len(blocks)},
            checksum=hashlib.sha256(full_text.encode()).hexdigest(),
        )

    def _get_body(self, message) -> str:
        if message.is_multipart():
            for part in message.walk():
                if part.get_content_type() == 'text/plain':
                    try:
                        return part.get_payload(decode=True).decode('utf-8', errors='replace')
                    except Exception:
                        return ''
        else:
            try:
                return message.get_payload(decode=True).decode('utf-8', errors='replace')
            except Exception:
                return ''

    def _stable_id(self, uri):
        import uuid
        return str(uuid.uuid5(uuid.NAMESPACE_URL, uri))

    def supports_incremental(self):
        return True

9.2 Code Ingestor

Python

# ingestors/code_ingestor.py

import ast
import re
from pathlib import Path
from datetime import datetime
import hashlib
from .base import BaseIngestor, Document, TextBlock

CODE_EXTENSIONS = {
    '.py': 'python', '.js': 'javascript', '.ts': 'typescript',
    '.go': 'go', '.rs': 'rust', '.java': 'java',
    '.cpp': 'cpp', '.c': 'c', '.rb': 'ruby', '.sh': 'bash',
}

class CodeIngestor(BaseIngestor):
    """
    Ingests source code files, extracting:
    - Module-level docstrings
    - Function/class docstrings
    - Inline comments
    - Function/class names (as entities)
    - Import statements (as relations to external concepts)
    """

    def can_handle(self, uri: str) -> bool:
        return Path(uri).suffix.lower() in CODE_EXTENSIONS

    def ingest(self, uri: str) -> Document:
        path = Path(uri)
        ext = path.suffix.lower()
        language = CODE_EXTENSIONS.get(ext, 'unknown')

        with open(uri, 'r', errors='replace') as f:
            source = f.read()

        blocks = []
        full_text_parts = []

        if language == 'python':
            blocks = self._extract_python(source, uri)
        else:
            blocks = self._extract_comments_generic(source, language)

        for block in blocks:
            full_text_parts.append(block.text)

        full_text = '\n\n'.join(full_text_parts)
        stat = path.stat()

        return Document(
            doc_id=self._stable_id(uri),
            source_type='code',
            uri=str(path.resolve()),
            title=path.name,
            author=None,
            created_at=datetime.fromtimestamp(stat.st_ctime),
            modified_at=datetime.fromtimestamp(stat.st_mtime),
            ingested_at=datetime.utcnow(),
            full_text=full_text,
            blocks=blocks,
            source_metadata={'language': language, 'file': str(path)},
            checksum=hashlib.sha256(full_text.encode()).hexdigest(),
        )

    def _extract_python(self, source: str, uri: str) -> list[TextBlock]:
        blocks = []
        try:
            tree = ast.parse(source)
            for node in ast.walk(tree):
                if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef, ast.ClassDef)):
                    docstring = ast.get_docstring(node)
                    if docstring:
                        blocks.append(TextBlock(
                            text=f"{node.name}: {docstring}",
                            block_type='paragraph',
                            offset=node.col_offset,
                            metadata={'kind': type(node).__name__, 'name': node.name},
                        ))
                elif isinstance(node, ast.Module):
                    docstring = ast.get_docstring(node)
                    if docstring:
                        blocks.append(TextBlock(
                            text=docstring,
                            block_type='paragraph',
                            offset=0,
                            metadata={'kind': 'module'},
                        ))
        except SyntaxError:
            # Fall back to comment extraction
            blocks = self._extract_comments_generic(source, 'python')
        return blocks

    def _extract_comments_generic(self, source: str, language: str) -> list[TextBlock]:
        comment_patterns = {
            'python': r'#\s*(.+)',
            'javascript': r'//\s*(.+)|/\*[\s\S]*?\*/',
            'typescript': r'//\s*(.+)|/\*[\s\S]*?\*/',
            'go': r'//\s*(.+)',
            'rust': r'///?\s*(.+)',
            'java': r'//\s*(.+)|/\*\*[\s\S]*?\*/',
        }
        pattern = comment_patterns.get(language, r'#\s*(.+)|//\s*(.+)')
        matches = re.finditer(pattern, source)
        blocks = []
        for match in matches:
            text = (match.group(1) or match.group(0)).strip()
            if len(text) > 10:
                blocks.append(TextBlock(
                    text=text,
                    block_type='paragraph',
                    offset=match.start(),
                ))
        return blocks

    def _stable_id(self, uri):
        import uuid
        return str(uuid.uuid5(uuid.NAMESPACE_URL, uri))

    def supports_incremental(self):
        return False

10. Extraction Pipeline Deep Dive
10.1 Pipeline Orchestrator

Python

# ingestion_pipeline.py

import logging
from datetime import datetime
from pathlib import Path
from typing import Optional

from ingestors.registry import IngestorRegistry
from extractor.chunker import Chunker
from extractor.entity_extractor import EntityExtractor
from extractor.concept_extractor import ConceptExtractor
from extractor.embedder import Embedder
from extractor.topic_modeler import TopicModeler
from extractor.relation_extractor import RelationExtractor
from graph.builder import GraphBuilder
from graph.merger import EntityMerger
from storage.document_repo import DocumentRepository
from storage.graph_repo import GraphRepository
from vector_store.store import VectorStore

logger = logging.getLogger("chronomap.pipeline")

class IngestionPipeline:
    def __init__(self, config: dict, doc_repo, graph_repo, vector_store, ws_manager=None):
        self.config = config
        self.doc_repo = doc_repo
        self.graph_repo = graph_repo
        self.vector_store = vector_store
        self.ws_manager = ws_manager  # WebSocket manager for real-time updates

        self.registry      = IngestorRegistry()
        self.chunker       = Chunker(
            chunk_size=config['extraction']['chunk_size_tokens'],
            overlap=config['extraction']['chunk_overlap_tokens'],
        )
        self.entity_extractor  = EntityExtractor(model=config['extraction']['spacy_model'])
        self.concept_extractor = ConceptExtractor(model_name=config['extraction']['embedding_model'])
        self.embedder          = Embedder()
        self.topic_modeler     = TopicModeler()
        self.relation_extractor = RelationExtractor()
        self.graph_builder     = GraphBuilder(repo=graph_repo, embedder=self.embedder)
        self.merger            = EntityMerger(repo=graph_repo, vector_store=vector_store)

        self._docs_since_refit = 0
        self._topic_refit_interval = config['extraction']['topic_refit_interval_docs']

    def run(self, uri: str) -> Optional[dict]:
        """
        Full ingestion pipeline for a single file.
        Returns summary dict: { doc_id, nodes_added, edges_added, chunks }
        """
        logger.info(f"Pipeline: starting {uri}")

        # 1. Select and run ingestor
        try:
            ingestor = self.registry.get_ingestor(uri)
        except ValueError:
            logger.warning(f"No ingestor for {uri}, skipping")
            return None

        doc = ingestor.ingest(uri)

        # 2. Check for changes (skip if unchanged)
        existing = self.doc_repo.get_by_uri(uri)
        if existing and existing.checksum == doc.checksum:
            logger.debug(f"No change detected for {uri}, skipping")
            return None

        # 3. Store document
        self.doc_repo.upsert(doc)
        if self.ws_manager:
            self.ws_manager.broadcast_sync({
                'type': 'ingest_started',
                'payload': {'doc_id': doc.doc_id, 'uri': uri, 'source_type': doc.source_type}
            })

        # 4. Chunk document
        chunks = self.chunker.chunk(doc)
        self.doc_repo.upsert_chunks(chunks)

        # 5. Embed chunks → upsert to vector store
        if chunks:
            embeddings = self.embedder.embed_chunks(chunks)
            for chunk, embedding in zip(chunks, embeddings):
                self.vector_store.upsert_chunk(
                    chunk_id=chunk.chunk_id,
                    doc_id=chunk.doc_id,
                    embedding=embedding,
                    text=chunk.text,
                    metadata={'source_type': doc.source_type},
                )

        # 6. Entity extraction
        all_entity_mentions = []
        for chunk in chunks:
            mentions = self.entity_extractor.extract(chunk)
            all_entity_mentions.extend(mentions)
        self.doc_repo.upsert_mentions(all_entity_mentions)

        # 7. Concept extraction (KeyBERT over full doc)
        concept_mentions = self.concept_extractor.extract(doc)

        # 8. Relation extraction
        all_triples = []
        for chunk in chunks:
            triples = self.relation_extractor.extract(chunk)
            all_triples.extend(triples)

        # 9. Topic assignment
        if self.config['extraction']['enable_topic_modeling']:
            self._assign_topics([doc])

        # 10. Build/update graph
        nodes_before = self.graph_repo.count_nodes()
        edges_before = self.graph_repo.count_edges()

        self.graph_builder.ingest_document(
            doc_id=doc.doc_id,
            entity_mentions=all_entity_mentions,
            concept_mentions=concept_mentions,
            relation_triples=all_triples,
            timestamp=doc.modified_at or doc.created_at or datetime.utcnow(),
        )

        # 11. Merge deduplication pass on newly-added nodes
        new_nodes = self.graph_repo.get_nodes_added_after(datetime.utcnow())
        for node in new_nodes[:50]:  # Limit merge pass per ingest
            candidates = self.merger.find_merge_candidates(node)
            for candidate_id, reason, confidence in candidates:
                if confidence >= 0.92:
                    self.merger.merge(node.node_id, candidate_id)

        nodes_added = self.graph_repo.count_nodes() - nodes_before
        edges_added = self.graph_repo.count_edges() - edges_before

        # 12. Notify UI
        summary = {
            'doc_id': doc.doc_id,
            'nodes_added': nodes_added,
            'edges_added': edges_added,
            'chunks': len(chunks),
        }
        if self.ws_manager:
            self.ws_manager.broadcast_sync({'type': 'ingest_complete', 'payload': summary})

        logger.info(f"Pipeline: complete {uri} → +{nodes_added} nodes, +{edges_added} edges")
        return summary

    def handle_deletion(self, uri: str):
        """Handle file deletion: mark document as deleted, prune orphan nodes."""
        doc = self.doc_repo.get_by_uri(uri)
        if not doc:
            return
        self.doc_repo.mark_deleted(doc.doc_id)
        self.vector_store.delete_by_doc(doc.doc_id)
        # Don't delete nodes immediately — they may appear in other documents
        # Instead, rebuild mention counts and let temporal staleness handle pruning
        self.graph_repo.rebuild_mention_counts_for_doc(doc.doc_id)

    def has_changed(self, uri: str) -> bool:
        """Return True if the file's checksum differs from last ingest."""
        existing = self.doc_repo.get_by_uri(uri)
        if not existing:
            return True
        from pathlib import Path
        import hashlib
        try:
            with open(uri, 'rb') as f:
                current_checksum = hashlib.sha256(f.read()).hexdigest()
            return existing.checksum != current_checksum
        except Exception:
            return True

    def _assign_topics(self, docs: list):
        self._docs_since_refit += len(docs)
        texts = [d.full_text[:10_000] for d in docs]
        if self._docs_since_refit >= self._topic_refit_interval:
            # Full refit with all documents in corpus
            all_texts = self.doc_repo.get_all_full_texts()
            topic_ids, topic_info = self.topic_modeler.fit_transform(all_texts)
            self._docs_since_refit = 0
        else:
            topic_ids = self.topic_modeler.transform(texts)
        # Store topic assignments...

11. Graph Layer Deep Dive
11.1 Edge Scorer

Python

# graph/scorer.py

import numpy as np
from typing import List

class EdgeScorer:
    """
    Recomputes edge weights using a combination of:
    1. Co-occurrence frequency (how often do A and B appear in the same doc?)
    2. Semantic similarity boost (if embeddings are close, weight is boosted)
    3. Recency (edges with recent last_seen get a small boost)
    4. Relation type bonus (explicit relation triples score higher than pure co-occurrence)
    """
    FREQ_WEIGHT = 0.5
    SEM_WEIGHT  = 0.3
    RECENCY_WEIGHT = 0.1
    RELATION_BONUS = 0.1

    def compute_weight(self, edge, node_a, node_b, max_freq: int) -> float:
        # Frequency component (normalized)
        freq_score = min(len(edge.doc_ids) / max(max_freq, 1), 1.0)

        # Semantic similarity component (cosine)
        sem_score = 0.0
        if node_a.embedding and node_b.embedding:
            a = np.array(node_a.embedding)
            b = np.array(node_b.embedding)
            sem_score = float(np.dot(a, b))  # Already normalized

        # Recency component
        from datetime import datetime
        days_since = (datetime.utcnow() - edge.last_seen).days
        recency_score = max(0.0, 1.0 - (days_since / 365))

        # Relation bonus
        relation_bonus = self.RELATION_BONUS if edge.edge_type == 'relation' else 0.0

        weight = (
            self.FREQ_WEIGHT    * freq_score +
            self.SEM_WEIGHT     * sem_score +
            self.RECENCY_WEIGHT * recency_score +
            relation_bonus
        )
        return min(weight, 1.0)

    def recompute_all_edges(self, repo):
        """Batch recompute edge weights for all edges. Run nightly."""
        edges = repo.get_all_edges()
        max_freq = max((len(e.doc_ids) for e in edges), default=1)
        for edge in edges:
            node_a = repo.get_node(edge.source_node_id)
            node_b = repo.get_node(edge.target_node_id)
            if not node_a or not node_b:
                continue
            new_weight = self.compute_weight(edge, node_a, node_b, max_freq)
            if abs(new_weight - edge.weight) > 0.01:
                edge.weight = new_weight
                repo.update_edge(edge)

11.2 Community Detection

Python

# graph/community.py

import networkx as nx
from typing import Dict, List

class CommunityDetector:
    """
    Detects communities (topic clusters) in the graph using the Louvain method.
    Used for:
    - ClusterView in UI (zoom-out shows community centroids)
    - Island detection (isolated communities)
    - Node coloring by community
    """

    def detect(self, graph: nx.Graph) -> Dict[str, int]:
        """
        Returns dict mapping node_id → community_id.
        Requires python-louvain (community) package.
        """
        try:
            import community as community_louvain
            partition = community_louvain.best_partition(graph, weight='weight')
            return partition
        except ImportError:
            # Fallback: use NetworkX greedy modularity communities
            communities = nx.community.greedy_modularity_communities(graph, weight='weight')
            partition = {}
            for i, community in enumerate(communities):
                for node_id in community:
                    partition[node_id] = i
            return partition

    def get_community_centroids(self, graph: nx.Graph, partition: Dict[str, int]) -> List[dict]:
        """
        For each community, find the highest-degree node as centroid.
        Used by ClusterView for zoom-out rendering.
        """
        from collections import defaultdict
        communities = defaultdict(list)
        for node_id, community_id in partition.items():
            communities[community_id].append(node_id)

        centroids = []
        for community_id, node_ids in communities.items():
            degrees = {nid: graph.degree(nid) for nid in node_ids}
            centroid_id = max(degrees, key=degrees.get)
            centroid_data = graph.nodes[centroid_id]
            centroids.append({
                'community_id': community_id,
                'centroid_node_id': centroid_id,
                'centroid_label': centroid_data.get('label', centroid_id),
                'node_count': len(node_ids),
                'node_ids': node_ids,
            })
        return sorted(centroids, key=lambda x: x['node_count'], reverse=True)

12. Temporal Layer Deep Dive
12.1 Snapshot Model

Python

# graph/temporal.py (extended)

class TemporalSnapshot:
    """
    Lightweight snapshot model.
    We do NOT store full graph copies per timestamp.
    Instead we store:
    - first_seen / last_seen on every node and edge (event model)
    - A timeline_keypoints table of significant moments
    
    graph_at(t) reconstructs any past state by filtering on these timestamps.
    """

    @staticmethod
    def record_keypoint(repo, timestamp, label: str):
        """Record a significant moment in the timeline (e.g., 'First 100 nodes')."""
        stats = {
            'node_count': repo.count_nodes(),
            'edge_count': repo.count_edges(),
            'doc_count': repo.count_documents(),
        }
        repo.insert_timeline_keypoint({
            'timestamp': timestamp.isoformat(),
            'label': label,
            **stats,
        })

12.2 Time Slider Logic (Frontend)

TypeScript

// lib/api.ts

export async function getGraphAtTimestamp(timestamp: string): Promise<GraphData> {
  const res = await fetch(
    `/api/v1/temporal/graph?timestamp=${encodeURIComponent(timestamp)}`
  );
  if (!res.ok) throw new Error('Failed to fetch temporal graph');
  return res.json();
}

export async function getGraphDiff(t1: string, t2: string): Promise<GraphDiff> {
  const res = await fetch(
    `/api/v1/temporal/diff?t1=${encodeURIComponent(t1)}&t2=${encodeURIComponent(t2)}`
  );
  return res.json();
}

13. Insight Engine Deep Dive
13.1 Insight Orchestrator

Python

# insight/orchestrator.py

import logging
from datetime import datetime
from .blind_spot_detector import BlindSpotDetector
from .island_detector import IslandDetector
from .obsession_tracker import ObsessionTracker
from .forgotten_idea_detector import ForgottenIdeaDetector

logger = logging.getLogger("chronomap.insights")

class InsightOrchestrator:
    def __init__(self, repo, vector_store, graph_temporal, config):
        self.repo = repo
        self.blind_spot = BlindSpotDetector(repo=repo, vector_store=vector_store)
        self.island     = IslandDetector(repo=repo)
        self.obsession  = ObsessionTracker(repo=repo)
        self.forgotten  = ForgottenIdeaDetector(repo=repo)
        self.temporal   = graph_temporal
        self.cache_ttl  = config['insights']['insight_cache_ttl_hours'] * 3600

    def refresh_all(self):
        """Run all insight detectors and cache results. Called on schedule."""
        logger.info("Refreshing all insights...")
        graph = self.temporal.graph_at(datetime.utcnow())

        insights = [
            ('blind_spot', self.blind_spot.detect(graph)),
            ('island',     self.island.detect(graph)),
            ('obsession',  self.obsession.detect()),
            ('forgotten',  self.forgotten.detect()),
        ]

        for insight_type, results in insights:
            self.repo.cache_insights(insight_type, results)
            logger.info(f"Cached {len(results)} {insight_type} insights")

    def get_blind_spots(self, limit=20, refresh=False):
        if refresh:
            graph = self.temporal.graph_at(datetime.utcnow())
            return self.blind_spot.detect(graph)[:limit]
        return self.repo.get_cached_insights('blind_spot', limit=limit)

    def get_islands(self, limit=10, refresh=False):
        if refresh:
            graph = self.temporal.graph_at(datetime.utcnow())
            return self.island.detect(graph)[:limit]
        return self.repo.get_cached_insights('island', limit=limit)

    def get_obsessions(self, limit=20, current_only=False):
        results = self.repo.get_cached_insights('obsession', limit=limit * 3)
        if current_only:
            results = [r for r in results if r.get('is_current')]
        return results[:limit]

    def get_forgotten(self, limit=20):
        return self.repo.get_cached_insights('forgotten', limit=limit)

14. UI & Visualization Deep Dive
14.1 Level-of-Detail System

TypeScript

// lib/lod.ts

/**
 * Level-of-detail: when graph has > MAX_VISIBLE_NODES nodes,
 * switch from individual node rendering to community centroid rendering.
 * 
 * Zoom levels:
 *   L0 (zoomed out): show only community centroids + inter-community edges
 *   L1 (medium):     show top-N nodes by mention_count + their edges
 *   L2 (zoomed in):  show full subgraph of selected node's N-hop neighborhood
 */

const MAX_VISIBLE_NODES = 2000;

export function selectVisibleNodes(
  nodes: KGNode[],
  edges: KGEdge[],
  zoomLevel: number,
  communities: CommunityInfo[],
  selectedNodeId?: string
): { nodes: KGNode[], edges: KGEdge[] } {
  
  if (nodes.length <= MAX_VISIBLE_NODES) {
    return { nodes, edges }; // No LOD needed
  }

  if (zoomLevel < 0.3) {
    // L0: community centroids only
    const centroidIds = new Set(communities.map(c => c.centroid_node_id));
    const visibleNodes = nodes.filter(n => centroidIds.has(n.node_id));
    const visibleEdges = edges.filter(e =>
      centroidIds.has(e.source_node_id) && centroidIds.has(e.target_node_id)
    );
    return { nodes: visibleNodes, edges: visibleEdges };
  }

  if (selectedNodeId) {
    // L2: neighborhood of selected node
    const neighborIds = new Set(
      edges
        .filter(e => e.source_node_id === selectedNodeId || e.target_node_id === selectedNodeId)
        .flatMap(e => [e.source_node_id, e.target_node_id])
    );
    neighborIds.add(selectedNodeId);
    const visibleNodes = nodes.filter(n => neighborIds.has(n.node_id));
    const visibleEdges = edges.filter(e =>
      neighborIds.has(e.source_node_id) && neighborIds.has(e.target_node_id)
    );
    return { nodes: visibleNodes, edges: visibleEdges };
  }

  // L1: top nodes by mention_count
  const topNodes = [...nodes]
    .sort((a, b) => b.mention_count - a.mention_count)
    .slice(0, MAX_VISIBLE_NODES);
  const topNodeIds = new Set(topNodes.map(n => n.node_id));
  const visibleEdges = edges.filter(e =>
    topNodeIds.has(e.source_node_id) && topNodeIds.has(e.target_node_id)
  );
  return { nodes: topNodes, edges: visibleEdges };
}

14.2 Svelte Store

TypeScript

// stores/graph.ts

import { writable, derived } from 'svelte/store';
import type { GraphData, KGNode, Insight } from '../types';

export const graphData = writable<GraphData | null>(null);
export const selectedNode = writable<KGNode | null>(null);
export const currentTimestamp = writable<string>(new Date().toISOString());
export const renderMode = writable<'3d' | '2d'>('3d');
export const searchQuery = writable<string>('');
export const activeInsightType = writable<string | null>(null);

// Derived: filter graph to highlighted nodes (e.g., blind spot pair)
export const highlightedNodeIds = writable<Set<string>>(new Set());

export const visibleNodeCount = derived(
  graphData,
  ($gd) => $gd?.nodes.length ?? 0
);

14.3 UI Layout (ASCII wireframe)

text

┌────────────────────────────────────────────────────────────────────┐
│  🧠 CHRONOMAP  │ 4,231 nodes  │ 18,402 edges  │ ⚙ Settings        │
├────────┬───────────────────────────────────────────────┬───────────┤
│ SEARCH │                                               │  NODE     │
│ [    ] │                                               │  DETAIL   │
│        │          3D GRAPH CANVAS                      │           │
│ FILTER │       (Three.js + D3 force)                   │  [label]  │
│ ○ All  │                                               │  Type: ORG│
│ ○ People│      ✦ privacy ─── ✦ surveillance           │           │
│ ○ Orgs │       \                /                      │  Mentions │
│ ○ Concepts│   ✦ GDPR ─── ✦ consent             ←BLIND │  42 times │
│        │                    \                   SPOT!  │           │
│ TOPICS │      ✦ machine ─── ✦ neural ─── ✦ AI        │  Sources  │
│ [list] │           learning      nets                  │  • doc1   │
│        │                                               │  • doc2   │
│ BLIND  │                                               │  • doc3   │
│ SPOTS  │                                               │           │
│ [list] │                                               │  Similar  │
├────────┴───────────────────────────────────────────────┴───────────┤
│  ◄──────────────────────────────────────────────────────────► NOW  │
│  Jan 2022          Jun 2023          Jan 2024          May 2025    │
│                  TIME SLIDER                              [Live]   │
└────────────────────────────────────────────────────────────────────┘

15. Search Deep Dive
15.1 Hybrid Search

Python

# api/routers/search.py

from fastapi import APIRouter
from pydantic import BaseModel
from typing import List, Optional
import numpy as np

router = APIRouter(prefix="/api/v1/search")

class SearchRequest(BaseModel):
    query: str
    top_k: int = 10
    search_type: str = "hybrid"  # "semantic" | "keyword" | "hybrid"
    filter_source_type: Optional[str] = None

class SearchResult(BaseModel):
    chunk_id: str
    doc_id: str
    doc_title: str
    source_type: str
    text_snippet: str
    score: float
    node_matches: list  # KGNodes mentioned in this chunk

@router.post("")
async def search(request: SearchRequest, services=Depends(get_services)):
    embedder    = services.embedder
    vector_store = services.vector_store
    doc_repo    = services.doc_repo
    graph_repo  = services.graph_repo

    results = []

    if request.search_type in ("semantic", "hybrid"):
        # Semantic: embed query → kNN in chunk vector space
        query_embedding = embedder.embed_query(request.query)
        semantic_hits = vector_store.query_chunks(
            embedding=query_embedding,
            top_k=request.top_k * 2,
        )
        for chunk_id, doc_id, score in semantic_hits:
            chunk = doc_repo.get_chunk(chunk_id)
            doc   = doc_repo.get(doc_id)
            if not chunk or not doc:
                continue
            # Find nodes mentioned in this chunk
            node_ids = doc_repo.get_nodes_in_chunk(chunk_id)
            nodes = [graph_repo.get_node(nid) for nid in node_ids]
            results.append(SearchResult(
                chunk_id=chunk_id,
                doc_id=doc_id,
                doc_title=doc.title,
                source_type=doc.source_type,
                text_snippet=chunk.text[:300],
                score=score,
                node_matches=[n for n in nodes if n],
            ))

    if request.search_type in ("keyword", "hybrid"):
        # Keyword: BM25-style full-text search via SQLite FTS5
        keyword_hits = doc_repo.fts_search(request.query, limit=request.top_k)
        # Merge with semantic results, deduplicate, re-rank
        existing_ids = {r.chunk_id for r in results}
        for hit in keyword_hits:
            if hit.chunk_id not in existing_ids:
                results.append(hit)

    # Re-rank hybrid results by combined score
    if request.search_type == "hybrid":
        results = sorted(results, key=lambda r: r.score, reverse=True)

    return {'results': results[:request.top_k], 'query': request.query}

16. Privacy & Security Model
16.1 Trust Model

text

┌─────────────────────────────────────┐
│  FULLY TRUSTED (localhost)          │
│  - FastAPI server (:8000)           │
│  - Svelte UI (served locally)       │
│  - SQLite database                  │
│  - Watch daemon                     │
│  - NLP/embedding workers            │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│  USER-CONTROLLED (filesystem)       │
│  - Watched directories              │
│  - Source files (read-only)         │
│  - Config and data directories      │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│  NEVER TOUCHED (network)            │
│  - No outbound calls ever           │
│  - No remote model APIs             │
│  - No telemetry, analytics, or CDN  │
└─────────────────────────────────────┘

16.2 API Localhost-Only Enforcement

Python

# api/middleware.py

from fastapi import Request, HTTPException
import ipaddress

async def localhost_only_middleware(request: Request, call_next):
    """Block any request not from localhost."""
    client_ip = request.client.host
    try:
        ip = ipaddress.ip_address(client_ip)
        if not (ip.is_loopback or ip.is_link_local):
            raise HTTPException(status_code=403, detail="ChronoMap API is localhost-only")
    except ValueError:
        raise HTTPException(status_code=403, detail="Invalid client IP")
    return await call_next(request)

16.3 Email Ingestor Privacy Safeguards

Email ingestion is disabled by default and requires explicit opt-in in config:

Python

# ingestors/email_ingestor.py (safety wrapper)

class EmailIngestor(BaseIngestor):
    def can_handle(self, uri: str) -> bool:
        if not self.config.get('ingestors', {}).get('email', {}).get('enabled', False):
            return False  # Never handle email unless explicitly enabled
        return Path(uri).suffix.lower() == '.mbox' or ...

16.4 Data Retention & Deletion

Python

# storage/document_repo.py

class DocumentRepository:
    def delete_all_data_for_source(self, source_type: str):
        """
        GDPR-style: delete all documents, chunks, mentions, and nodes
        exclusively from a given source type. Nodes that appear in
        other source types are preserved.
        """
        # 1. Get all doc_ids for source_type
        doc_ids = self.db.execute(
            "SELECT doc_id FROM documents WHERE source_type = ?", (source_type,)
        ).fetchall()
        
        for (doc_id,) in doc_ids:
            # 2. Delete chunks and their vectors
            self.db.execute("DELETE FROM chunks WHERE doc_id = ?", (doc_id,))
            self.vector_store.delete_by_doc(doc_id)
            
            # 3. Rebuild node mention counts
            self.rebuild_mention_counts_for_doc(doc_id)
        
        # 4. Delete documents
        self.db.execute("DELETE FROM documents WHERE source_type = ?", (source_type,))
        self.db.commit()

    def purge_all(self):
        """Nuclear option: delete entire ChronoMap database."""
        tables = ['mentions', 'edges', 'nodes', 'chunks', 'documents',
                  'topics', 'document_topics', 'insights', 'mention_timeseries']
        for table in tables:
            self.db.execute(f"DELETE FROM {table}")
        self.db.commit()
        # Also clear vector store
        self.vector_store.clear_all()

17. Storage & Persistence
17.1 Database Connection

Python

# storage/db.py

import sqlite3
import threading
from pathlib import Path

class Database:
    def __init__(self, db_path: str):
        self.db_path = db_path
        self._local = threading.local()
        Path(db_path).parent.mkdir(parents=True, exist_ok=True)
        self._run_migrations()

    def _get_conn(self) -> sqlite3.Connection:
        """Thread-local connection (SQLite is not thread-safe for writes)."""
        if not hasattr(self._local, 'conn') or self._local.conn is None:
            self._local.conn = sqlite3.connect(
                self.db_path,
                check_same_thread=False,
                timeout=30.0,
            )
            self._local.conn.row_factory = sqlite3.Row
            # Performance pragmas
            self._local.conn.executescript("""
                PRAGMA journal_mode = WAL;
                PRAGMA synchronous = NORMAL;
                PRAGMA temp_store = MEMORY;
                PRAGMA mmap_size = 134217728;  -- 128MB
                PRAGMA cache_size = -64000;     -- 64MB cache
                PRAGMA busy_timeout = 10000;
            """)
        return self._local.conn

    @property
    def conn(self) -> sqlite3.Connection:
        return self._get_conn()

    def execute(self, sql: str, params=()) -> sqlite3.Cursor:
        return self.conn.execute(sql, params)

    def executemany(self, sql: str, params_list) -> sqlite3.Cursor:
        return self.conn.executemany(sql, params_list)

    def commit(self):
        self.conn.commit()

    def _run_migrations(self):
        from .migrations import MigrationRunner
        MigrationRunner(self.db_path).run()

17.2 Graph Repository

Python

# storage/graph_repo.py

import json
from typing import Optional, List
from datetime import datetime
from .db import Database
from ..graph.types import KGNode, KGEdge

class GraphRepository:
    def __init__(self, db: Database):
        self.db = db

    def insert_node(self, node: KGNode):
        self.db.execute("""
            INSERT INTO nodes 
            (node_id, label, normalized_label, node_type, entity_label,
             first_seen, last_seen, mention_count, doc_ids, embedding, updated_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            node.node_id, node.label, node.label.lower().strip(),
            node.node_type, node.entity_label,
            node.first_seen.isoformat(), node.last_seen.isoformat(),
            node.mention_count,
            json.dumps(node.doc_ids),
            bytes(b for b in self._serialize_embedding(node.embedding)) if node.embedding else None,
            datetime.utcnow().isoformat(),
        ))
        self.db.commit()

    def get_node(self, node_id: str) -> Optional[KGNode]:
        row = self.db.execute(
            "SELECT * FROM nodes WHERE node_id = ? AND is_merged = 0", (node_id,)
        ).fetchone()
        if not row:
            return None
        return self._row_to_node(row)

    def get_nodes_at(self, timestamp: datetime, staleness_days: int) -> List[KGNode]:
        """Return nodes that existed at the given timestamp."""
        staleness_cutoff = f"-{staleness_days} days"
        rows = self.db.execute("""
            SELECT * FROM nodes
            WHERE first_seen <= ?
              AND (last_seen >= datetime(?, ?) OR last_seen >= ?)
              AND is_merged = 0
        """, (
            timestamp.isoformat(),
            timestamp.isoformat(), staleness_cutoff,
            timestamp.isoformat(),
        )).fetchall()
        return [self._row_to_node(r) for r in rows]

    def get_edges_at(self, timestamp: datetime, staleness_days: int) -> List[KGEdge]:
        staleness_cutoff = f"-{staleness_days} days"
        rows = self.db.execute("""
            SELECT * FROM edges
            WHERE first_seen <= ?
              AND (last_seen >= datetime(?, ?) OR last_seen >= ?)
        """, (
            timestamp.isoformat(),
            timestamp.isoformat(), staleness_cutoff,
            timestamp.isoformat(),
        )).fetchall()
        return [self._row_to_edge(r) for r in rows]

    def find_nodes_by_normalized_label(self, normalized_label: str) -> List[KGNode]:
        rows = self.db.execute(
            "SELECT * FROM nodes WHERE normalized_label = ? AND is_merged = 0",
            (normalized_label,)
        ).fetchall()
        return [self._row_to_node(r) for r in rows]

    def redirect_edges(self, from_node_id: str, to_node_id: str):
        """Redirect all edges from old node to canonical node."""
        self.db.execute(
            "UPDATE edges SET source_node_id = ? WHERE source_node_id = ?",
            (to_node_id, from_node_id)
        )
        self.db.execute(
            "UPDATE edges SET target_node_id = ? WHERE target_node_id = ?",
            (to_node_id, from_node_id)
        )
        # Remove any self-loops created by merge
        self.db.execute(
            "DELETE FROM edges WHERE source_node_id = target_node_id"
        )
        self.db.commit()

    def _row_to_node(self, row) -> KGNode:
        return KGNode(
            node_id=row['node_id'],
            label=row['label'],
            node_type=row['node_type'],
            entity_label=row['entity_label'],
            first_seen=datetime.fromisoformat(row['first_seen']),
            last_seen=datetime.fromisoformat(row['last_seen']),
            mention_count=row['mention_count'],
            doc_ids=json.loads(row['doc_ids']),
            embedding=self._deserialize_embedding(row['embedding']) if row['embedding'] else None,
        )

    def _serialize_embedding(self, embedding) -> bytes:
        import struct
        return struct.pack(f"{len(embedding)}f", *embedding)

    def _deserialize_embedding(self, data: bytes) -> list:
        import struct
        n = len(data) // 4
        return list(struct.unpack(f"{n}f", data))

18. Logging, Observability & Debugging
18.1 Structured Logging

Python

# config/logging_setup.py

import logging
import logging.handlers
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        return json.dumps({
            'ts': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'module': record.module,
            'line': record.lineno,
            **(record.__dict__.get('extra', {})),
        })

def setup_logging(config: dict):
    log_level = config['logging']['level'].upper()
    log_file  = config['logging']['file']
    
    handler_file = logging.handlers.RotatingFileHandler(
        log_file,
        maxBytes=config['logging']['max_size_mb'] * 1024 * 1024,
        backupCount=config['logging']['max_backups'],
    )
    handler_file.setFormatter(JSONFormatter())

    handler_console = logging.StreamHandler()
    handler_console.setFormatter(logging.Formatter(
        '%(asctime)s [%(levelname)s] %(name)s: %(message)s'
    ))

    logging.basicConfig(
        level=getattr(logging, log_level),
        handlers=[handler_file, handler_console],
    )

18.2 Ingestion Pipeline Metrics

Python

# ingestion_pipeline.py (metrics)

from dataclasses import dataclass, field
from typing import List

@dataclass
class PipelineMetrics:
    total_files_processed: int = 0
    total_files_skipped: int = 0
    total_files_errored: int = 0
    total_nodes_created: int = 0
    total_edges_created: int = 0
    total_chunks_created: int = 0
    errors: List[dict] = field(default_factory=list)

    def record_error(self, uri: str, error: Exception):
        self.total_files_errored += 1
        self.errors.append({
            'uri': uri,
            'error': str(error),
            'error_type': type(error).__name__,
        })

    def to_dict(self) -> dict:
        return {
            'files_processed': self.total_files_processed,
            'files_skipped': self.total_files_skipped,
            'files_errored': self.total_files_errored,
            'nodes_created': self.total_nodes_created,
            'edges_created': self.total_edges_created,
            'chunks_created': self.total_chunks_created,
            'recent_errors': self.errors[-10:],
        }

19. Testing Strategy
19.1 Test Pyramid

text

                   ┌────────────────┐
                   │   E2E (5%)     │
                   │  Playwright    │
                   │  (UI + API)    │
                   └───────┬────────┘
            ┌──────────────┴──────────────┐
            │    Integration (25%)         │
            │  Pipeline + DB + Vector      │
            │  API endpoints               │
            └──────────────┬──────────────┘
     ┌────────────────────┴───────────────────┐
     │           Unit Tests (70%)              │
     │  Ingestors | Extractors | Graph Logic   │
     │  Insight Detectors | Chunker            │
     └─────────────────────────────────────────┘

19.2 Unit Tests

Python

# tests/unit/test_chunker.py

import pytest
from extractor.chunker import Chunker

def test_chunker_respects_chunk_size():
    chunker = Chunker(chunk_size=100, overlap=20)
    text = " ".join(["word"] * 500)
    
    class FakeDoc:
        doc_id = "test_doc"
        full_text = text
    
    chunks = chunker.chunk(FakeDoc())
    assert len(chunks) > 1
    for chunk in chunks:
        assert len(chunk.text.split()) <= 120  # Approximate token count

def test_chunker_overlap():
    chunker = Chunker(chunk_size=50, overlap=10)
    # Verify that consecutive chunks share ~10 tokens of content
    # ...

def test_chunk_ids_are_stable():
    """Same input always produces same chunk IDs."""
    chunker = Chunker()
    class FakeDoc:
        doc_id = "test_doc"
        full_text = "This is a test document with some content."
    
    chunks1 = chunker.chunk(FakeDoc())
    chunks2 = chunker.chunk(FakeDoc())
    assert [c.chunk_id for c in chunks1] == [c.chunk_id for c in chunks2]

Python

# tests/unit/test_graph_builder.py

import pytest
from unittest.mock import MagicMock
from datetime import datetime
from graph.builder import GraphBuilder
from extractor.entity_extractor import EntityMention
from extractor.concept_extractor import ConceptMention

def test_node_upsert_increments_mention_count(mock_repo, mock_embedder):
    builder = GraphBuilder(repo=mock_repo, embedder=mock_embedder)
    
    mention = EntityMention(
        mention_id="m1", doc_id="doc1", chunk_id="c1",
        text="Alan Turing", label="PERSON", start_char=0, end_char=11, confidence=0.95
    )
    
    # First ingest
    builder.ingest_document("doc1", [mention], [], [], datetime.utcnow())
    assert mock_repo.insert_node.called
    
    # Second ingest — same entity
    existing_node = MagicMock()
    existing_node.mention_count = 1
    existing_node.doc_ids = ["doc1"]
    mock_repo.get_node.return_value = existing_node
    
    builder.ingest_document("doc2", [mention], [], [], datetime.utcnow())
    assert mock_repo.update_node.called
    assert existing_node.mention_count == 2

def test_edge_created_for_cooccurrence(mock_repo, mock_embedder):
    builder = GraphBuilder(repo=mock_repo, embedder=mock_embedder)
    mention_a = EntityMention(mention_id="m1", doc_id="doc1", chunk_id="c1",
        text="Alice", label="PERSON", start_char=0, end_char=5, confidence=1.0)
    mention_b = EntityMention(mention_id="m2", doc_id="doc1", chunk_id="c1",
        text="Bob", label="PERSON", start_char=10, end_char=13, confidence=1.0)
    
    mock_repo.get_node.return_value = None  # Both new nodes
    mock_repo.get_edge.return_value = None  # New edge
    
    builder.ingest_document("doc1", [mention_a, mention_b], [], [], datetime.utcnow())
    assert mock_repo.insert_edge.called

Python

# tests/unit/test_blind_spot_detector.py

import pytest
import networkx as nx
import numpy as np
from unittest.mock import MagicMock
from insight.blind_spot_detector import BlindSpotDetector

def test_detects_semantically_similar_disconnected_nodes():
    # Create a graph with two disconnected clusters
    G = nx.Graph()
    G.add_nodes_from(['A', 'B', 'C', 'X', 'Y', 'Z'],
        label='node', node_type='concept', entity_label='CONCEPT', mention_count=5)
    G.add_edges_from([('A','B'), ('B','C'), ('X','Y'), ('Y','Z')])
    # A and X are semantically similar but in different components (no path)

    mock_repo = MagicMock()
    mock_vector_store = MagicMock()

    # A and X have similar embeddings
    node_a = MagicMock(); node_a.embedding = np.random.randn(384).tolist(); node_a.node_type = 'concept'
    node_x = MagicMock(); node_x.embedding = node_a.embedding  # Identical = similarity 1.0
    
    mock_repo.get_node.side_effect = lambda nid: node_a if nid == 'A' else node_x
    mock_vector_store.query_nodes.return_value = [('X', 0.98)]

    detector = BlindSpotDetector(repo=mock_repo, vector_store=mock_vector_store)
    blind_spots = detector.detect(G)

    assert len(blind_spots) > 0
    pair_labels = [(b.node_a_id, b.node_b_id) for b in blind_spots]
    assert ('A', 'X') in pair_labels or ('X', 'A') in pair_labels

19.3 Integration Tests

Python

# tests/integration/test_ingestion_pipeline.py

import pytest
import tempfile
from pathlib import Path
from ingestion_pipeline import IngestionPipeline

SAMPLE_MARKDOWN = """---
title: Test Note
created: 2024-01-15
---

# Machine Learning and Privacy

This note explores the intersection of machine learning and data privacy.
Alan Turing proposed early frameworks for computation that now underpin AI.
The GDPR regulation creates obligations for organizations handling personal data.

## Key Concepts

- Neural networks require vast amounts of training data
- Differential privacy offers mathematical guarantees
- Federated learning keeps data local while training models
"""

@pytest.fixture
def tmp_workspace():
    with tempfile.TemporaryDirectory() as tmp:
        yield Path(tmp)

def test_end_to_end_markdown_ingestion(tmp_workspace, test_db, test_vector_store):
    # Write sample markdown
    md_file = tmp_workspace / "test_note.md"
    md_file.write_text(SAMPLE_MARKDOWN)

    pipeline = IngestionPipeline(
        config=test_config(),
        doc_repo=test_db.doc_repo,
        graph_repo=test_db.graph_repo,
        vector_store=test_vector_store,
    )

    result = pipeline.run(str(md_file))

    assert result is not None
    assert result['chunks'] > 0
    assert result['nodes_added'] > 0

    # Verify document stored
    doc = test_db.doc_repo.get_by_uri(str(md_file))
    assert doc is not None
    assert doc.title == "Test Note"
    assert doc.source_type == "markdown"

    # Verify entities extracted
    nodes = test_db.graph_repo.get_all_nodes()
    labels = [n.label for n in nodes]
    # Alan Turing and GDPR should be found as entities
    assert any("Turing" in label or "Alan" in label for label in labels)

    # Verify vector search works
    query_emb = pipeline.embedder.embed_query("machine learning privacy")
    hits = test_vector_store.query_chunks(query_emb, top_k=5)
    assert len(hits) > 0

def test_deduplication_merges_aliases(tmp_workspace, test_db, test_vector_store):
    """Verify that 'Alan Turing' and 'Turing' merge to the same canonical node."""
    # Create two files with different surface forms of same entity
    file1 = tmp_workspace / "file1.md"
    file1.write_text("Alan Turing was a mathematician.")
    file2 = tmp_workspace / "file2.md"
    file2.write_text("Turing proposed the Turing test.")
    
    pipeline = IngestionPipeline(config=test_config(), ...)
    pipeline.run(str(file1))
    pipeline.run(str(file2))

    # After merge pass, there should be one canonical node, not two
    nodes = [n for n in test_db.graph_repo.get_all_nodes()
             if 'turing' in n.label.lower() and not n.is_merged]
    assert len(nodes) == 1

20. Build, Packaging & Installation
20.1 Makefile

Makefile

.PHONY: all install dev test lint clean build-ui setup-models

# Install Python dependencies
install:
	pip install -e ".[all]"
	cd ui && npm install

# Build Svelte UI
build-ui:
	cd ui && npm run build
	cp -r ui/dist/* api/static/

# Download NLP models
setup-models:
	python -m spacy download en_core_web_sm
	python -m spacy download en_core_web_trf
	python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"

# Full build
all: install build-ui setup-models

# Dev mode: API with hot reload + UI dev server
dev:
	uvicorn cmd.chronomap.main:app --reload --host 127.0.0.1 --port 8000 &
	cd ui && npm run dev

# Run tests
test:
	pytest tests/unit/ -v --cov=. --cov-report=term-missing

test-integration:
	pytest tests/integration/ -v --tb=short

test-all:
	pytest tests/ -v --cov=.

# Lint
lint:
	ruff check .
	mypy . --ignore-missing-imports
	cd ui && npm run check

# Clean generated files
clean:
	rm -rf ui/dist api/static/ __pycache__ .pytest_cache .coverage dist/
	find . -name "*.pyc" -delete

# Initialize ChronoMap home directory
init:
	python -c "from config.config_loader import load_config; load_config()"
	@echo "ChronoMap initialized at ~/.chronomap/"

# Package as single executable (PyInstaller)
package:
	pip install pyinstaller
	pyinstaller --onefile --name chronomap \
	  --add-data "ui/dist:api/static" \
	  --add-data "config/default_config.toml:config" \
	  cmd/chronomap/main.py

# Cross-platform release
release-linux:
	docker run --rm -v $(PWD):/src python:3.11-slim \
	  bash -c "cd /src && pip install pyinstaller && make package"

release-macos:
	make package
	mv dist/chronomap dist/chronomap-macos-$(shell uname -m)

20.2 pyproject.toml

toml

[project]
name = "chronomap"
version = "1.0.0"
description = "Local-first, self-building personal knowledge graph"
requires-python = ">=3.11"

dependencies = [
    # API
    "fastapi>=0.110.0",
    "uvicorn[standard]>=0.27.0",
    "pydantic>=2.0.0",
    "python-multipart>=0.0.9",

    # NLP
    "spacy>=3.7.0",
    "keybert>=0.8.0",
    "sentence-transformers>=2.6.0",
    "bertopic>=0.16.0",

    # Graph
    "networkx>=3.2.0",

    # Storage
    "sqlite-vec>=0.1.0",

    # Ingestion
    "PyMuPDF>=1.23.0",         # PDF
    "python-frontmatter>=1.1.0", # Markdown frontmatter
    "icalendar>=5.0.0",         # Calendar ICS

    # Filesystem watching
    "watchdog>=4.0.0",

    # Config
    "tomli>=2.0.0 ; python_version < '3.11'",  # stdlib tomllib in 3.11+

    # Utilities
    "numpy>=1.26.0",
    "tqdm>=4.66.0",
    "python-dateutil>=2.8.0",
    "aiofiles>=23.0.0",
]

[project.optional-dependencies]
chroma = ["chromadb>=0.4.0"]
email = []
dev = [
    "pytest>=8.0.0",
    "pytest-cov>=5.0.0",
    "pytest-asyncio>=0.23.0",
    "ruff>=0.3.0",
    "mypy>=1.9.0",
    "httpx>=0.27.0",   # For FastAPI test client
]

[project.scripts]
chronomap = "cmd.chronomap.main:cli"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

20.3 Docker Compose

YAML

# docker-compose.yml

version: '3.8'
services:
  chronomap:
    build: .
    ports:
      - "127.0.0.1:8000:8000"
    volumes:
      # Mount user's data sources (read-only)
      - ~/Documents:/data/documents:ro
      - ~/Notes:/data/notes:ro
      - ~/Downloads:/data/downloads:ro
      # Ch
