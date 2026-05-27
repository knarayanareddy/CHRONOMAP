🧠 CHRONOMAP
Comprehensive Engineering Design Document

Version 1.0 | Local-First, Self-Building Neural Knowledge Graph of Your Life
TABLE OF CONTENTS

    Project Overview & Vision
    Goals, Non-Goals & Constraints
    System Architecture
    Module Breakdown
        4.1 Ingestion Layer
        4.2 Watch Daemon (Continuous Ingestion)
        4.3 Extraction Layer (NLP Pipeline)
        4.4 Graph Layer (Knowledge Graph Builder)
        4.5 Temporal Layer (Time-Travel Engine)
        4.6 Insight Engine
        4.7 Vector Search Layer
        4.8 REST API Server
        4.9 Frontend Graph UI
    Data Models & Schemas
    API Specifications
    Directory Structure
    Configuration System
    Ingestion Source Registry
    NLP Pipeline Deep Dive
    Graph Model Deep Dive
    Temporal Engine Deep Dive
    Insight Engine Deep Dive
    Vector Search Deep Dive
    Privacy & Local-First Model
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

ChronoMap is a local-first, passively self-building visual knowledge graph that runs entirely on the user's machine. It sits between the user's life of digital artifacts and their understanding of their own thinking, acting as an intelligent cartographer that:

    Watches everything the user touches — notes, PDFs, browser history, email, code, calendar events, ebook highlights — and ingests it automatically
    Extracts entities, concepts, and relationships using local NLP models (no cloud, no API keys)
    Builds and continuously evolves a personal knowledge graph stored in local SQLite + a vector index
    Renders an interactive, force-directed neural map where nodes are concepts and edges are relationships, colored by topic cluster, weighted by connection strength
    Supports time-travel: scrub a timeline slider to see your knowledge graph as it existed at any past moment
    Surfaces insights: blind spots (semantically related ideas you never connected), knowledge islands (isolated topic clusters), obsessions (concepts dominating your attention over time), and forgotten ideas (high-relevance nodes that went dormant)

All processing is local. No data leaves the machine. No AI company sees your thoughts.
1.2 The Problem Being Solved

Knowledge workers and curious people accumulate vast quantities of digital artifacts — notes, bookmarks, PDFs, highlights, code comments, emails — spread across dozens of apps and formats. These artifacts represent the actual shape of a person's thinking, but:

    No tool connects them automatically. Obsidian, Roam, and Logseq require you to manually create [[links]]. Your knowledge graph is only as connected as you explicitly made it.
    Most connections are implicit. You don't link your 2022 PDF annotation to your 2024 code comment about the same concept — but the relationship is real and valuable.
    Time is invisible. There's no tool that lets you ask "what was I obsessed with in Q3?" or "when did I first encounter this idea?" or "what have I forgotten?"
    Blind spots are invisible by definition. You can't search for what you don't know you know.

ChronoMap's bet is that the real knowledge graph is already there — it just needs to be extracted, connected, and made visible.
1.3 Design Philosophy
Principle	Description
Passive-first	The graph builds itself. Zero manual linking required.
Local-first	All inference, storage, and rendering on the user's machine. No cloud.
Temporal	Every node and edge has a birth time. The graph is a living timeline, not a static snapshot.
Insight-driven	The graph's value is in what it reveals, not just what it shows. Blind spots, islands, obsessions.
Composable	Plugin ingestors for new source types. Community NLP extractors.
Privacy by design	The system stores the minimum viable data. Graph data never leaves the machine.
Transparent	Every node and edge has provenance: what document, at what time, with what confidence.
2. Goals, Non-Goals & Constraints
2.1 Goals (In Scope)

    Automatic ingestion of Markdown notes, PDFs, browser history/bookmarks, local email archives (.mbox/.eml), source code files, ebook highlights (Kindle/Kobo exports), and .ics calendar files
    Filesystem watch daemon for real-time incremental re-ingestion on file change
    Local NLP extraction pipeline: named entity recognition (spaCy), keyword/keyphrase extraction (KeyBERT), semantic embeddings (SentenceTransformers), topic modeling (BERTopic), relation extraction (dependency parsing)
    SQLite-backed knowledge graph with nodes (entities/concepts), edges (relations/co-mentions), and full provenance (source document, span, timestamp)
    Vector index for semantic similarity search (sqlite-vec or Chroma)
    Time-travel API: graph_at(timestamp) returning the graph state at any past moment
    Insight engine: blind spot detection, island detection, obsession tracker, forgotten idea surfacer
    FastAPI REST backend serving graph, search, temporal, and insight endpoints
    Svelte frontend with interactive force-directed graph (D3 + Three.js), timeline slider, node detail panel, search bar, cluster view, and insight panels
    Cross-platform: macOS, Linux, Windows

2.2 Non-Goals (Explicitly Out of Scope)

    ❌ Cloud sync or remote storage of any kind
    ❌ Integration with cloud-based note apps (Notion, Evernote) — only local files
    ❌ Editing notes or documents from within ChronoMap
    ❌ OCR of scanned documents (image-only PDFs)
    ❌ Real-time collaboration or sharing of graphs
    ❌ Any outbound network calls for NLP inference (all models local)
    ❌ Mobile client (v1.0 is desktop-only)
    ❌ Manual link creation (the graph is entirely automatic)

2.3 Constraints

    All NLP inference must run locally (spaCy, SentenceTransformers, KeyBERT, BERTopic via local model weights)
    SQLite is the only database engine (no Postgres, no Mongo)
    The watch daemon must support inotify (Linux), kqueue (macOS), and ReadDirectoryChangesW (Windows) via the watchdog Python library
    The graph renderer must handle up to 50,000 nodes and 200,000 edges at cluster-level zoom without browser freeze (LOD required)
    Re-ingestion of a changed file must complete in < 5 seconds for files under 1MB
    Semantic search must return results in < 500ms P99 for a corpus of 100,000 chunks

3. System Architecture
3.1 High-Level Architecture Diagram

text

┌────────────────────────────────────────────────────────────────────────┐
│                           USER'S MACHINE                               │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                     CHRONOMAP CORE (Python)                      │  │
│  │                                                                  │  │
│  │  ┌──────────────┐    ┌────────────────┐    ┌─────────────────┐  │  │
│  │  │  Watch Daemon │    │ Ingestion Layer│    │ Extraction Layer│  │  │
│  │  │  (watchdog)  │───►│ (per-source    │───►│ (spaCy/KeyBERT/ │  │  │
│  │  │              │    │  ingestors)    │    │  SentenceXfmrs) │  │  │
│  │  └──────────────┘    └────────────────┘    └────────┬────────┘  │  │
│  │                                                      │           │  │
│  │  ┌─────────────────────────────────────────────────▼─────────┐  │  │
│  │  │                     GRAPH LAYER                            │  │  │
│  │  │  builder.py ──► merger.py ──► scorer.py ──► temporal.py   │  │  │
│  │  └──────────────────────────┬─────────────────────────────── ┘  │  │
│  │                             │                                     │  │
│  │  ┌──────────────────────────▼──────────────────────────────────┐ │  │
│  │  │               STORAGE LAYER                                  │ │  │
│  │  │   SQLite (documents, chunks, entities, edges, mentions)      │ │  │
│  │  │   sqlite-vec / Chroma  (vector index for embeddings)         │ │  │
│  │  └──────────────────────────┬───────────────────────────────── ┘ │  │
│  │                             │                                     │  │
│  │  ┌───────────────┐   ┌──────▼──────────┐   ┌──────────────────┐ │  │
│  │  │ Insight Engine│   │ FastAPI REST API │   │ Vector Search    │ │  │
│  │  │ (blind spots, │◄──│  Port 7331      │──►│  (kNN retrieval) │ │  │
│  │  │  islands,     │   │                 │   │                  │ │  │
│  │  │  obsessions)  │   └──────┬──────────┘   └──────────────────┘ │  │
│  │  └───────────────┘          │                                    │  │
│  └───────────────────────────  │  ────────────────────────────────┘  │
│                                │                                       │
│  ┌─────────────────────────────▼─────────────────────────────────┐   │
│  │                   SVELTE FRONTEND  Port 5173                   │   │
│  │   GraphCanvas (D3+Three.js)  │  TimeSlider  │  NodeDetail      │   │
│  │   SearchBar  │  ClusterView  │  BlindSpotPanel │ ObsessionTracker│  │
│  └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘

3.2 Data Flow (Step by Step)

text

Step 1:  Watch Daemon detects new or changed file in watched directories
Step 2:  File dispatched to matching Ingestor (by extension/source type)
Step 3:  Ingestor normalizes file → list of Document records (doc_id, source,
         uri, title, created_at, modified_at, raw_text_chunks[])
Step 4:  Each chunk passed through Extraction Pipeline:
         a) spaCy NER → entity mentions (PERSON, ORG, GPE, CONCEPT...)
         b) KeyBERT   → top-k keyphrases per chunk
         c) SentenceTransformers → dense embedding vector per chunk
         d) BERTopic  → topic assignment per chunk (batch, async)
         e) Dependency parsing → relation triples per chunk
Step 5:  Graph Builder receives extracted records:
         - Upsert entity/concept nodes (by canonical ID)
         - Merger deduplicates nodes (string norm + embedding similarity)
         - Scorer computes edge weights (co-mention freq + cosine sim)
         - Temporal layer stamps first_seen / last_seen on nodes and edges
Step 6:  Embeddings written to vector index (sqlite-vec or Chroma)
Step 7:  Graph persisted to SQLite (nodes, edges, mentions, provenance tables)
Step 8:  FastAPI server reflects updated graph on next client poll or push
Step 9:  Insight Engine runs async after each ingestion batch:
         - Blind spot detector (kNN + path distance check)
         - Island detector (connected components)
         - Obsession tracker (mention density over time windows)
         - Forgotten idea surfacer (high-centrality nodes gone dormant)
Step 10: Frontend polls /graph or receives WebSocket push → re-renders

3.3 Component Ownership
Component	Language	Owns
Watch Daemon	Python (watchdog)	File change detection, dispatch
Ingestion Layer	Python	Source parsing, normalization, chunking
Extraction Layer	Python (spaCy, KeyBERT, SentenceTransformers, BERTopic)	NLP entity/concept/relation/embedding extraction
Graph Layer	Python (NetworkX + SQLite)	Node/edge build, merge, score, temporal
Vector Search	Python (sqlite-vec / Chroma)	kNN embedding retrieval
Insight Engine	Python	Blind spots, islands, obsessions, forgotten ideas
API Server	Python (FastAPI)	REST + WebSocket API
Frontend	TypeScript (Svelte + D3 + Three.js)	Graph render, UI, search
4. Module Breakdown
4.1 Ingestion Layer

Purpose

The Ingestion Layer transforms raw user artifacts from heterogeneous sources into a uniform internal Document record, chunked and ready for the extraction pipeline.

Supported Sources (v1.0)
Source	Ingestor	File Types	Notes
Markdown Notes	markdown_ingestor.py	.md, .mdx	Frontmatter parsed for metadata
PDFs	pdf_ingestor.py	.pdf	Text + highlights via PyMuPDF
Browser Bookmarks	browser_ingestor.py	bookmarks.html, bookmarks.json	Chrome/Firefox/Safari export
Browser History	browser_ingestor.py	History (SQLite), places.sqlite	Page title + URL + visit time
Local Email	email_ingestor.py	.mbox, .eml	Subject + body + sender/recipient
Source Code	code_ingestor.py	.py, .ts, .go, .rs, .js, .md	File-level + docstring/comment extraction
Ebook Highlights	ebook_ingestor.py	My Clippings.txt (Kindle), Kobo SQLite	Highlight text + book/author metadata
Calendar	calendar_ingestor.py	.ics	Event title + description + attendees + datetime

Normalized Document Record

Python

# ingestors/base.py

from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional

@dataclass
class TextChunk:
    chunk_id: str
    doc_id: str
    text: str
    chunk_index: int
    char_start: int
    char_end: int
    page_number: Optional[int] = None      # PDFs
    line_number: Optional[int] = None      # Code
    metadata: dict = field(default_factory=dict)

@dataclass
class Document:
    doc_id: str
    source_type: str            # "markdown" | "pdf" | "browser_bookmark" | ...
    uri: str                    # File path or URL
    title: str
    created_at: datetime
    modified_at: datetime
    author: Optional[str]
    tags: list[str]
    chunks: list[TextChunk]
    raw_metadata: dict = field(default_factory=dict)

class BaseIngestor:
    """All ingestors must implement this interface."""

    def can_handle(self, path: str) -> bool:
        raise NotImplementedError

    def ingest(self, path: str) -> list[Document]:
        raise NotImplementedError

    def _chunk_text(self, text: str, doc_id: str, chunk_size: int = 512,
                    overlap: int = 64) -> list[TextChunk]:
        """Sliding window chunker with overlap to preserve context at boundaries."""
        chunks = []
        words = text.split()
        step = chunk_size - overlap
        for i, start in enumerate(range(0, len(words), step)):
            chunk_words = words[start:start + chunk_size]
            chunk_text = " ".join(chunk_words)
            char_start = len(" ".join(words[:start]))
            chunks.append(TextChunk(
                chunk_id=f"{doc_id}_chunk_{i}",
                doc_id=doc_id,
                text=chunk_text,
                chunk_index=i,
                char_start=char_start,
                char_end=char_start + len(chunk_text),
            ))
        return chunks

Markdown Ingestor

Python

# ingestors/markdown_ingestor.py

import re
import yaml
from pathlib import Path
from datetime import datetime
from .base import BaseIngestor, Document

class MarkdownIngestor(BaseIngestor):

    FRONTMATTER_RE = re.compile(r'^---\s*\n(.*?)\n---\s*\n', re.DOTALL)

    def can_handle(self, path: str) -> bool:
        return Path(path).suffix.lower() in {'.md', '.mdx'}

    def ingest(self, path: str) -> list[Document]:
        p = Path(path)
        raw = p.read_text(encoding='utf-8', errors='replace')

        # Parse YAML frontmatter
        metadata = {}
        body = raw
        match = self.FRONTMATTER_RE.match(raw)
        if match:
            try:
                metadata = yaml.safe_load(match.group(1)) or {}
            except yaml.YAMLError:
                pass
            body = raw[match.end():]

        stat = p.stat()
        doc_id = f"md_{p.stem}_{int(stat.st_mtime)}"

        doc = Document(
            doc_id=doc_id,
            source_type="markdown",
            uri=str(p.resolve()),
            title=metadata.get("title", p.stem),
            created_at=datetime.fromtimestamp(stat.st_ctime),
            modified_at=datetime.fromtimestamp(stat.st_mtime),
            author=metadata.get("author"),
            tags=metadata.get("tags", []),
            chunks=self._chunk_text(body, doc_id),
            raw_metadata=metadata,
        )
        return [doc]

PDF Ingestor (with Highlight Extraction)

Python

# ingestors/pdf_ingestor.py

import fitz  # PyMuPDF
import hashlib
from pathlib import Path
from datetime import datetime
from .base import BaseIngestor, Document, TextChunk

class PDFIngestor(BaseIngestor):

    def can_handle(self, path: str) -> bool:
        return Path(path).suffix.lower() == '.pdf'

    def ingest(self, path: str) -> list[Document]:
        p = Path(path)
        doc_id = f"pdf_{hashlib.md5(str(p).encode()).hexdigest()[:12]}"
        pdf = fitz.open(str(p))

        full_text_chunks = []
        highlight_chunks = []

        for page_num, page in enumerate(pdf):
            # Standard text extraction
            text = page.get_text("text")
            for chunk in self._chunk_text(text, doc_id, chunk_size=400):
                chunk.page_number = page_num + 1
                chunk.metadata["chunk_type"] = "body"
                full_text_chunks.append(chunk)

            # Annotation/highlight extraction
            for annot in page.annots():
                if annot.type[0] == 8:  # Highlight annotation
                    rects = annot.vertices
                    highlight_text = page.get_text("text", clip=annot.rect).strip()
                    if highlight_text:
                        highlight_chunks.append(TextChunk(
                            chunk_id=f"{doc_id}_highlight_{page_num}_{len(highlight_chunks)}",
                            doc_id=doc_id,
                            text=highlight_text,
                            chunk_index=len(full_text_chunks) + len(highlight_chunks),
                            char_start=0,
                            char_end=len(highlight_text),
                            page_number=page_num + 1,
                            metadata={"chunk_type": "highlight", "color": str(annot.colors)},
                        ))

        meta = pdf.metadata
        return [Document(
            doc_id=doc_id,
            source_type="pdf",
            uri=str(p.resolve()),
            title=meta.get("title", p.stem),
            created_at=datetime.fromtimestamp(p.stat().st_ctime),
            modified_at=datetime.fromtimestamp(p.stat().st_mtime),
            author=meta.get("author"),
            tags=[],
            chunks=full_text_chunks + highlight_chunks,
            raw_metadata=dict(meta),
        )]

4.2 Watch Daemon (Continuous Ingestion)

Purpose

The Watch Daemon monitors user-configured directories for file system events (create, modify, delete) and dispatches changed files to the appropriate ingestor. It is the heartbeat that keeps ChronoMap "alive."

Architecture

text

User Configured Paths (config.toml)
       │
       ▼
watchdog Observer (inotify / kqueue / FSEvents / ReadDirChanges)
       │ FileCreatedEvent / FileModifiedEvent / FileDeletedEvent
       ▼
ChangeHandler.dispatch()
       │
       ├─ Debounce (100ms) → prevents duplicate events on rapid saves
       │
       ├─ Extension filter → match to registered Ingestor
       │
       ├─ Hash check → skip if file content unchanged (same SHA256)
       │
       └─► IngestorRegistry.process(path)
                  │
                  ▼
            ExtractionPipeline.process(document)
                  │
                  ▼
            GraphBuilder.update()

Implementation

Python

# daemon/watch_daemon.py

import hashlib
import threading
import time
from pathlib import Path
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler, FileModifiedEvent, FileCreatedEvent

class ChronoMapDaemon:
    def __init__(self, config, ingestor_registry, pipeline):
        self.config = config
        self.registry = ingestor_registry
        self.pipeline = pipeline
        self.observer = Observer()
        self._file_hashes: dict[str, str] = {}
        self._debounce_timers: dict[str, threading.Timer] = {}
        self._lock = threading.Lock()

    def start(self):
        handler = self._build_handler()
        for watch_path in self.config.watch_paths:
            self.observer.schedule(handler, watch_path, recursive=True)
        self.observer.start()

    def stop(self):
        self.observer.stop()
        self.observer.join()

    def _build_handler(self):
        daemon = self

        class ChangeHandler(FileSystemEventHandler):
            def on_modified(self, event):
                if not event.is_directory:
                    daemon._schedule_process(event.src_path)

            def on_created(self, event):
                if not event.is_directory:
                    daemon._schedule_process(event.src_path)

            def on_deleted(self, event):
                if not event.is_directory:
                    daemon._handle_deletion(event.src_path)

        return ChangeHandler()

    def _schedule_process(self, path: str):
        """Debounce: wait 100ms after last event before processing."""
        with self._lock:
            if path in self._debounce_timers:
                self._debounce_timers[path].cancel()
            timer = threading.Timer(0.1, self._process, args=[path])
            self._debounce_timers[path] = timer
            timer.start()

    def _process(self, path: str):
        try:
            content_hash = self._file_hash(path)
            with self._lock:
                if self._file_hashes.get(path) == content_hash:
                    return  # Content unchanged — skip
                self._file_hashes[path] = content_hash

            ingestor = self.registry.find(path)
            if ingestor is None:
                return  # Unsupported file type

            documents = ingestor.ingest(path)
            for doc in documents:
                self.pipeline.process(doc)
        except Exception as e:
            # Daemon errors must never crash — log and continue
            import logging
            logging.getLogger(__name__).error(f"Error processing {path}: {e}")

    def _handle_deletion(self, path: str):
        # Mark all entities/edges from this source as last_seen = now
        self.pipeline.handle_source_deletion(path)

    @staticmethod
    def _file_hash(path: str) -> str:
        h = hashlib.sha256()
        with open(path, 'rb') as f:
            for chunk in iter(lambda: f.read(65536), b''):
                h.update(chunk)
        return h.hexdigest()

4.3 Extraction Layer (NLP Pipeline)

Purpose

The Extraction Layer is ChronoMap's intelligence core. It reads Document records from the ingestion layer and produces structured extractions: entities, concepts, embeddings, topic assignments, and relation triples.

Pipeline Overview

text

TextChunk
    │
    ├──► spaCy NER          → EntityMention(span, label, doc_id, chunk_id)
    │
    ├──► KeyBERT            → ConceptMention(phrase, score, doc_id, chunk_id)
    │
    ├──► SentenceTransformers → Embedding(vector[384], chunk_id)
    │
    ├──► BERTopic (batch)   → TopicAssignment(topic_id, label, chunk_id)
    │
    └──► Dependency Parser  → RelationTriple(subject, predicate, object, chunk_id)

Entity Extractor (spaCy)

Python

# extractor/entity_extractor.py

import spacy
from dataclasses import dataclass
from typing import Optional

@dataclass
class EntityMention:
    mention_id: str
    doc_id: str
    chunk_id: str
    text: str               # Surface form ("Alan Turing")
    label: str              # spaCy NER label ("PERSON", "ORG", "GPE", etc.)
    start_char: int
    end_char: int
    canonical_id: Optional[str] = None  # Set by merger after dedup

class EntityExtractor:

    # ChronoMap-relevant entity types (filter out noise)
    RELEVANT_LABELS = {
        "PERSON", "ORG", "GPE", "LOC", "PRODUCT", "EVENT",
        "WORK_OF_ART", "LAW", "LANGUAGE", "DATE", "NORP"
    }

    def __init__(self, model: str = "en_core_web_trf"):
        self.nlp = spacy.load(model)

    def extract(self, chunk) -> list[EntityMention]:
        doc = self.nlp(chunk.text)
        mentions = []
        for ent in doc.ents:
            if ent.label_ not in self.RELEVANT_LABELS:
                continue
            mentions.append(EntityMention(
                mention_id=f"{chunk.chunk_id}_ent_{ent.start_char}",
                doc_id=chunk.doc_id,
                chunk_id=chunk.chunk_id,
                text=ent.text,
                label=ent.label_,
                start_char=ent.start_char,
                end_char=ent.end_char,
            ))
        return mentions

Concept Extractor (KeyBERT)

Python

# extractor/concept_extractor.py

from keybert import KeyBERT
from dataclasses import dataclass

@dataclass
class ConceptMention:
    mention_id: str
    doc_id: str
    chunk_id: str
    phrase: str
    score: float            # KeyBERT relevance score 0.0-1.0
    ngram_range: tuple

class ConceptExtractor:

    def __init__(self, model: str = "all-MiniLM-L6-v2"):
        self.kw_model = KeyBERT(model=model)

    def extract(self, chunk, top_n: int = 8) -> list[ConceptMention]:
        keywords = self.kw_model.extract_keywords(
            chunk.text,
            keyphrase_ngram_range=(1, 3),
            stop_words='english',
            top_n=top_n,
            use_mmr=True,           # Maximal marginal relevance for diversity
            diversity=0.5,
        )
        return [
            ConceptMention(
                mention_id=f"{chunk.chunk_id}_kw_{i}",
                doc_id=chunk.doc_id,
                chunk_id=chunk.chunk_id,
                phrase=phrase,
                score=score,
                ngram_range=(1, 3),
            )
            for i, (phrase, score) in enumerate(keywords)
            if score >= 0.2  # Drop very low-confidence concepts
        ]

Embedder (SentenceTransformers)

Python

# extractor/embedder.py

import numpy as np
from sentence_transformers import SentenceTransformer
from dataclasses import dataclass

@dataclass
class ChunkEmbedding:
    chunk_id: str
    doc_id: str
    vector: np.ndarray      # Shape: (384,) for all-MiniLM-L6-v2
    model_name: str

class Embedder:

    DEFAULT_MODEL = "sentence-transformers/all-MiniLM-L6-v2"

    def __init__(self, model_name: str = DEFAULT_MODEL, batch_size: int = 64):
        self.model = SentenceTransformer(model_name)
        self.model_name = model_name
        self.batch_size = batch_size

    def embed_chunks(self, chunks: list) -> list[ChunkEmbedding]:
        texts = [c.text for c in chunks]
        # Batch encode for efficiency
        vectors = self.model.encode(
            texts,
            batch_size=self.batch_size,
            show_progress_bar=False,
            normalize_embeddings=True,   # L2 normalize for cosine similarity via dot product
        )
        return [
            ChunkEmbedding(
                chunk_id=chunk.chunk_id,
                doc_id=chunk.doc_id,
                vector=vector,
                model_name=self.model_name,
            )
            for chunk, vector in zip(chunks, vectors)
        ]

    def embed_query(self, query: str) -> np.ndarray:
        return self.model.encode(query, normalize_embeddings=True)

Relation Extractor (Dependency Parsing)

Python

# extractor/relation_extractor.py

import spacy
from dataclasses import dataclass

@dataclass
class RelationTriple:
    triple_id: str
    doc_id: str
    chunk_id: str
    subject: str            # Entity/concept surface form
    predicate: str          # Verb/relation
    obj: str                # Entity/concept surface form
    confidence: float

class RelationExtractor:

    def __init__(self, nlp):
        self.nlp = nlp      # Re-use the spaCy model from EntityExtractor

    def extract(self, chunk) -> list[RelationTriple]:
        doc = self.nlp(chunk.text)
        triples = []

        for token in doc:
            # Simple SVO extraction: subject → verb → object
            if token.dep_ in ("nsubj", "nsubjpass") and token.head.pos_ == "VERB":
                subject = token.text
                predicate = token.head.lemma_
                # Find direct objects of this verb
                for child in token.head.children:
                    if child.dep_ in ("dobj", "attr", "prep"):
                        obj = child.text
                        triples.append(RelationTriple(
                            triple_id=f"{chunk.chunk_id}_rel_{len(triples)}",
                            doc_id=chunk.doc_id,
                            chunk_id=chunk.chunk_id,
                            subject=subject,
                            predicate=predicate,
                            obj=obj,
                            confidence=0.6,  # Dependency parse baseline confidence
                        ))

        return triples

Topic Modeler (BERTopic)

Python

# extractor/topic_modeler.py

from bertopic import BERTopic
from dataclasses import dataclass
import numpy as np

@dataclass
class TopicAssignment:
    chunk_id: str
    doc_id: str
    topic_id: int
    topic_label: str        # BERTopic auto-generated label e.g. "machine_learning_neural"
    probability: float

class TopicModeler:
    """
    BERTopic runs in batch mode over the full corpus.
    It is NOT called per-chunk but invoked periodically
    (on startup + after large ingestion batches).
    """

    def __init__(self, min_topic_size: int = 10):
        self.model = BERTopic(
            min_topic_size=min_topic_size,
            calculate_probabilities=True,
            verbose=False,
        )
        self._fitted = False

    def fit_transform(self, chunks: list, embeddings: np.ndarray) -> list[TopicAssignment]:
        texts = [c.text for c in chunks]
        topics, probs = self.model.fit_transform(texts, embeddings)
        self._fitted = True

        assignments = []
        topic_info = self.model.get_topic_info()
        topic_labels = {row["Topic"]: row["Name"] for _, row in topic_info.iterrows()}

        for chunk, topic_id, prob_vec in zip(chunks, topics, probs):
            prob = float(prob_vec[topic_id]) if topic_id >= 0 else 0.0
            assignments.append(TopicAssignment(
                chunk_id=chunk.chunk_id,
                doc_id=chunk.doc_id,
                topic_id=int(topic_id),
                topic_label=topic_labels.get(topic_id, "outlier"),
                probability=prob,
            ))
        return assignments

    def transform(self, chunks: list, embeddings: np.ndarray) -> list[TopicAssignment]:
        """Use already-fitted model for incremental assignment."""
        if not self._fitted:
            return self.fit_transform(chunks, embeddings)
        texts = [c.text for c in chunks]
        topics, probs = self.model.transform(texts, embeddings)
        # (same mapping as fit_transform)
        ...

4.4 Graph Layer (Knowledge Graph Builder)

Purpose

The Graph Layer receives all extracted records and assembles them into a queryable knowledge graph stored in SQLite, with NetworkX used for in-memory computation (community detection, path queries, centrality).

builder.py

Python

# graph/builder.py

import networkx as nx
from datetime import datetime
from storage.db import GraphDB

class GraphBuilder:

    def __init__(self, db: GraphDB, merger, scorer):
        self.db = db
        self.merger = merger
        self.scorer = scorer
        self.G = nx.Graph()   # In-memory working graph (loaded from DB on startup)

    def process_extractions(self, doc_id: str, entities: list,
                            concepts: list, triples: list, timestamp: datetime):
        """
        Main entry point called by the extraction pipeline.
        Upserts all extracted nodes/edges into the graph.
        """
        # 1. Upsert entity nodes
        for mention in entities:
            canonical_id = self.merger.resolve_entity(mention)
            self.db.upsert_node(
                node_id=canonical_id,
                label=mention.label,
                surface_form=mention.text,
                node_type="entity",
                first_seen=timestamp,
            )
            self.db.add_mention(
                node_id=canonical_id,
                doc_id=doc_id,
                chunk_id=mention.chunk_id,
                timestamp=timestamp,
            )
            self.G.add_node(canonical_id, label=mention.label, node_type="entity")

        # 2. Upsert concept nodes
        for concept_mention in concepts:
            canonical_id = self.merger.resolve_concept(concept_mention)
            self.db.upsert_node(
                node_id=canonical_id,
                label="CONCEPT",
                surface_form=concept_mention.phrase,
                node_type="concept",
                first_seen=timestamp,
            )
            self.db.add_mention(
                node_id=canonical_id,
                doc_id=doc_id,
                chunk_id=concept_mention.chunk_id,
                timestamp=timestamp,
            )
            self.G.add_node(canonical_id, label="CONCEPT", node_type="concept")

        # 3. Build co-mention edges within same chunk
        all_node_ids = [
            self.merger.resolve_entity(e) for e in entities
        ] + [
            self.merger.resolve_concept(c) for c in concepts
        ]
        self._add_comention_edges(all_node_ids, doc_id, timestamp)

        # 4. Add typed relation edges from triples
        for triple in triples:
            self._add_relation_edge(triple, timestamp)

        # 5. Recompute edge weights for affected edges
        self.scorer.rescore_edges(self.G, self.db, list(set(all_node_ids)))

    def _add_comention_edges(self, node_ids: list[str], doc_id: str, timestamp: datetime):
        seen = set()
        for i, n1 in enumerate(node_ids):
            for n2 in node_ids[i+1:]:
                if n1 == n2:
                    continue
                key = tuple(sorted([n1, n2]))
                if key in seen:
                    continue
                seen.add(key)
                self.db.upsert_edge(
                    src=n1, dst=n2,
                    edge_type="co_mention",
                    doc_id=doc_id,
                    timestamp=timestamp,
                )
                self.G.add_edge(n1, n2, edge_type="co_mention")

    def _add_relation_edge(self, triple, timestamp: datetime):
        src_id = self.merger.resolve_surface(triple.subject)
        dst_id = self.merger.resolve_surface(triple.obj)
        if src_id and dst_id:
            self.db.upsert_edge(
                src=src_id, dst=dst_id,
                edge_type=triple.predicate,
                doc_id=triple.doc_id,
                timestamp=timestamp,
            )
            self.G.add_edge(src_id, dst_id, edge_type=triple.predicate)

merger.py — Entity Deduplication

Python

# graph/merger.py

import re
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

class NodeMerger:
    """
    Layered entity resolution:
    1. String normalization (lowercase, strip punctuation, stemming)
    2. Embedding similarity (cosine > threshold → same node)
    3. Context overlap (co-mentioned with the same nodes → likely same)
    """

    MERGE_SIMILARITY_THRESHOLD = 0.92

    def __init__(self, db, embedder):
        self.db = db
        self.embedder = embedder
        self._surface_to_id: dict[str, str] = {}   # Cache

    def resolve_entity(self, mention) -> str:
        normalized = self._normalize(mention.text)

        # 1. Exact normalized match
        if normalized in self._surface_to_id:
            return self._surface_to_id[normalized]

        # 2. Embedding similarity against existing nodes of same label
        candidates = self.db.get_nodes_by_label(mention.label, limit=200)
        if candidates:
            candidate_embeddings = np.array([c["embedding"] for c in candidates])
            query_emb = self.embedder.embed_query(mention.text).reshape(1, -1)
            sims = cosine_similarity(query_emb, candidate_embeddings)[0]
            best_idx = np.argmax(sims)
            if sims[best_idx] >= self.MERGE_SIMILARITY_THRESHOLD:
                canonical_id = candidates[best_idx]["node_id"]
                self._surface_to_id[normalized] = canonical_id
                return canonical_id

        # 3. New node — mint a new canonical ID
        import hashlib, uuid
        canonical_id = f"node_{hashlib.md5(normalized.encode()).hexdigest()[:12]}"
        self._surface_to_id[normalized] = canonical_id
        return canonical_id

    def resolve_concept(self, concept_mention) -> str:
        return self.resolve_surface(concept_mention.phrase)

    def resolve_surface(self, surface: str) -> str | None:
        if not surface or not surface.strip():
            return None
        normalized = self._normalize(surface)
        return self._surface_to_id.get(normalized) or \
               f"node_{__import__('hashlib').md5(normalized.encode()).hexdigest()[:12]}"

    @staticmethod
    def _normalize(text: str) -> str:
        text = text.lower().strip()
        text = re.sub(r"[^\w\s]", "", text)
        text = re.sub(r"\s+", " ", text)
        return text

scorer.py — Edge Weighting

Python

# graph/scorer.py

import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

class EdgeScorer:
    """
    Edge weight = α × co-mention frequency + β × semantic similarity
    Default: α=0.6, β=0.4
    """

    def __init__(self, db, embedder, alpha: float = 0.6, beta: float = 0.4):
        self.db = db
        self.embedder = embedder
        self.alpha = alpha
        self.beta = beta

    def rescore_edges(self, G, db, affected_node_ids: list[str]):
        for node_id in affected_node_ids:
            neighbors = list(G.neighbors(node_id))
            for neighbor_id in neighbors:
                edge_data = db.get_edge(node_id, neighbor_id)
                if edge_data is None:
                    continue

                co_mention_count = edge_data["co_mention_count"]
                max_count = db.get_max_co_mention_count()
                freq_score = co_mention_count / max(max_count, 1)

                # Get representative embeddings for both nodes
                emb_a = db.get_node_embedding(node_id)
                emb_b = db.get_node_embedding(neighbor_id)

                if emb_a is not None and emb_b is not None:
                    sim_score = float(cosine_similarity(
                        emb_a.reshape(1, -1), emb_b.reshape(1, -1)
                    )[0][0])
                else:
                    sim_score = 0.0

                weight = self.alpha * freq_score + self.beta * sim_score
                G[node_id][neighbor_id]["weight"] = weight
                db.update_edge_weight(node_id, neighbor_id, weight)

4.5 Temporal Layer (Time-Travel Engine)

Purpose

Every node and edge in the graph carries first_seen and last_seen timestamps. The temporal layer enables querying the exact state of the graph at any historical moment.

temporal.py

Python

# graph/temporal.py

import networkx as nx
from datetime import datetime
from storage.db import GraphDB

class TemporalGraph:
    """
    Provides a time-slice view of the knowledge graph.
    graph_at(t) returns a NetworkX graph containing only
    nodes and edges that existed at time t.
    """

    def __init__(self, db: GraphDB):
        self.db = db

    def graph_at(self, timestamp: datetime) -> nx.Graph:
        """Return the knowledge graph as it existed at `timestamp`."""
        G = nx.Graph()

        # Nodes that had first_seen <= timestamp AND (last_seen IS NULL OR last_seen >= timestamp)
        nodes = self.db.query("""
            SELECT node_id, label, surface_form, node_type, first_seen, last_seen
            FROM nodes
            WHERE first_seen <= ?
              AND (last_seen IS NULL OR last_seen >= ?)
        """, (timestamp, timestamp))

        for node in nodes:
            G.add_node(
                node["node_id"],
                label=node["label"],
                surface_form=node["surface_form"],
                node_type=node["node_type"],
                first_seen=node["first_seen"],
            )

        # Edges active at timestamp
        edges = self.db.query("""
            SELECT src, dst, edge_type, weight, first_seen, co_mention_count
            FROM edges
            WHERE first_seen <= ?
              AND (last_seen IS NULL OR last_seen >= ?)
              AND src IN (SELECT node_id FROM nodes WHERE first_seen <= ?)
              AND dst IN (SELECT node_id FROM nodes WHERE first_seen <= ?)
        """, (timestamp, timestamp, timestamp, timestamp))

        for edge in edges:
            G.add_edge(
                edge["src"], edge["dst"],
                edge_type=edge["edge_type"],
                weight=edge["weight"],
                first_seen=edge["first_seen"],
            )

        return G

    def node_timeline(self, node_id: str) -> list[dict]:
        """Return per-month mention counts for a node — powers the obsession tracker."""
        return self.db.query("""
            SELECT
                strftime('%Y-%m', timestamp) AS month,
                COUNT(*) AS mention_count
            FROM mentions
            WHERE node_id = ?
            GROUP BY month
            ORDER BY month ASC
        """, (node_id,))

    def graph_diff(self, t1: datetime, t2: datetime) -> dict:
        """Return nodes/edges added and removed between two timestamps."""
        G1 = self.graph_at(t1)
        G2 = self.graph_at(t2)
        return {
            "nodes_added":   list(set(G2.nodes) - set(G1.nodes)),
            "nodes_removed": list(set(G1.nodes) - set(G2.nodes)),
            "edges_added":   list(set(G2.edges) - set(G1.edges)),
            "edges_removed": list(set(G1.edges) - set(G2.edges)),
        }

4.6 Insight Engine

Purpose

The Insight Engine is ChronoMap's highest-value feature. It continuously analyzes the graph structure and surfaces actionable insights about the user's knowledge: what they're missing, what's disconnected, what they're obsessed with, and what they've forgotten.

blind_spot_detector.py

Python

# graph/blind_spot_detector.py

import numpy as np
from dataclasses import dataclass

@dataclass
class BlindSpot:
    node_a_id: str
    node_a_surface: str
    node_b_id: str
    node_b_surface: str
    semantic_similarity: float
    graph_distance: int         # -1 if no path exists
    score: float                # Higher = stronger blind spot
    supporting_docs: list[str]  # Doc IDs where each concept appears

class BlindSpotDetector:
    """
    A blind spot is a pair of concepts that are semantically
    related (high embedding cosine similarity) but structurally
    disconnected (large graph distance or no path).

    SCALABLE APPROACH:
    Use vector kNN index to find semantic neighbors per node,
    then only check graph distance for those pairs.
    This is O(n × k) not O(n²).
    """

    SIMILARITY_THRESHOLD = 0.75
    MIN_GRAPH_DISTANCE = 4      # Pairs closer than this are "already connected enough"
    TOP_K_NEIGHBORS = 20

    def __init__(self, db, vector_index, G):
        self.db = db
        self.vector_index = vector_index
        self.G = G

    def detect(self, max_results: int = 50) -> list[BlindSpot]:
        blind_spots = []
        nodes = list(self.G.nodes(data=True))

        for node_id, node_data in nodes:
            node_embedding = self.db.get_node_embedding(node_id)
            if node_embedding is None:
                continue

            # Step 1: Find semantic neighbors via kNN index
            neighbors = self.vector_index.query(
                node_embedding,
                k=self.TOP_K_NEIGHBORS + 1,  # +1 because self is in results
            )

            for neighbor_id, similarity in neighbors:
                if neighbor_id == node_id:
                    continue
                if similarity < self.SIMILARITY_THRESHOLD:
                    continue
                if not self.G.has_node(neighbor_id):
                    continue

                # Step 2: Check graph distance (only for semantically similar pairs)
                try:
                    path_length = nx.shortest_path_length(self.G, node_id, neighbor_id)
                except nx.NetworkXNoPath:
                    path_length = -1  # No path at all = maximum structural separation

                if path_length == -1 or path_length >= self.MIN_GRAPH_DISTANCE:
                    score = similarity * (1.0 if path_length == -1 else 1.0 / path_length)
                    blind_spots.append(BlindSpot(
                        node_a_id=node_id,
                        node_a_surface=node_data.get("surface_form", node_id),
                        node_b_id=neighbor_id,
                        node_b_surface=self.G.nodes[neighbor_id].get("surface_form", neighbor_id),
                        semantic_similarity=float(similarity),
                        graph_distance=path_length,
                        score=score,
                        supporting_docs=self._get_supporting_docs(node_id, neighbor_id),
                    ))

        # Deduplicate (A,B) and (B,A) pairs, sort by score descending
        seen = set()
        deduped = []
        for bs in sorted(blind_spots, key=lambda x: x.score, reverse=True):
            key = tuple(sorted([bs.node_a_id, bs.node_b_id]))
            if key not in seen:
                seen.add(key)
                deduped.append(bs)

        return deduped[:max_results]

    def _get_supporting_docs(self, node_a_id: str, node_b_id: str) -> list[str]:
        docs_a = set(self.db.get_doc_ids_for_node(node_a_id))
        docs_b = set(self.db.get_doc_ids_for_node(node_b_id))
        return list(docs_a | docs_b)[:5]

island_detector.py

Python

# graph/island_detector.py

import networkx as nx
from dataclasses import dataclass

@dataclass
class KnowledgeIsland:
    island_id: int
    node_ids: list[str]
    node_count: int
    top_concepts: list[str]         # Most central nodes in the island
    topic_labels: list[str]         # BERTopic labels for this island's nodes
    internal_density: float         # Edge density within the island

class IslandDetector:
    """
    Identifies isolated knowledge clusters: groups of nodes that
    are internally well-connected but have few or no bridges
    to the rest of the graph.
    """

    MIN_ISLAND_SIZE = 3
    MAX_BRIDGE_EDGES = 2            # Islands with <= this many cross-edges are "isolated"

    def __init__(self, G, db):
        self.G = G
        self.db = db

    def detect(self) -> list[KnowledgeIsland]:
        # Use community detection (Louvain) to find clusters
        try:
            import community as community_louvain
            partition = community_louvain.best_partition(self.G)
        except ImportError:
            # Fallback: connected components
            partition = {}
            for i, component in enumerate(nx.connected_components(self.G)):
                for node in component:
                    partition[node] = i

        # Group nodes by community
        communities: dict[int, list[str]] = {}
        for node, comm_id in partition.items():
            communities.setdefault(comm_id, []).append(node)

        islands = []
        for comm_id, node_ids in communities.items():
            if len(node_ids) < self.MIN_ISLAND_SIZE:
                continue

            subgraph = self.G.subgraph(node_ids)

            # Count bridges to outside
            bridge_count = sum(
                1 for n in node_ids
                for neighbor in self.G.neighbors(n)
                if neighbor not in set(node_ids)
            )

            if bridge_count <= self.MAX_BRIDGE_EDGES:
                centrality = nx.degree_centrality(subgraph)
                top_nodes = sorted(centrality, key=centrality.get, reverse=True)[:5]
                top_concepts = [
                    self.G.nodes[n].get("surface_form", n) for n in top_nodes
                ]
                density = nx.density(subgraph)
                topic_labels = self.db.get_topic_labels_for_nodes(node_ids)

                islands.append(KnowledgeIsland(
                    island_id=comm_id,
                    node_ids=node_ids,
                    node_count=len(node_ids),
                    top_concepts=top_concepts,
                    topic_labels=list(set(topic_labels))[:5],
                    internal_density=density,
                ))

        return sorted(islands, key=lambda x: x.node_count, reverse=True)

obsession_tracker.py

Python

# graph/obsession_tracker.py

from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class ObsessionEntry:
    node_id: str
    surface_form: str
    node_type: str
    window_mention_count: int
    all_time_mention_count: int
    trend: str          # "rising" | "stable" | "falling"
    first_seen: datetime
    last_seen: datetime
    time_series: list[dict]   # [{"month": "2024-01", "count": 12}, ...]

class ObsessionTracker:
    """
    Identifies concepts dominating the user's attention
    in a configurable time window (default: 30 days).
    Also detects trend (rising vs falling vs stable).
    """

    def __init__(self, db):
        self.db = db

    def get_obsessions(self, window_days: int = 30,
                       top_n: int = 20) -> list[ObsessionEntry]:
        window_start = datetime.utcnow() - timedelta(days=window_days)
        prior_start = window_start - timedelta(days=window_days)

        # Current window mention counts
        current = self.db.query("""
            SELECT node_id, COUNT(*) as cnt
            FROM mentions
            WHERE timestamp >= ?
            GROUP BY node_id
            ORDER BY cnt DESC
            LIMIT ?
        """, (window_start, top_n * 2))

        # Prior window mention counts (for trend detection)
        prior_map = {
            row["node_id"]: row["cnt"]
            for row in self.db.query("""
                SELECT node_id, COUNT(*) as cnt
                FROM mentions
                WHERE timestamp >= ? AND timestamp < ?
                GROUP BY node_id
            """, (prior_start, window_start))
        }

        results = []
        for row in current[:top_n]:
            node = self.db.get_node(row["node_id"])
            prior_count = prior_map.get(row["node_id"], 0)
            current_count = row["cnt"]

            if current_count > prior_count * 1.3:
                trend = "rising"
            elif current_count < prior_count * 0.7:
                trend = "falling"
            else:
                trend = "stable"

            time_series = self.db.query("""
                SELECT strftime('%Y-%m', timestamp) as month, COUNT(*) as count
                FROM mentions WHERE node_id = ?
                GROUP BY month ORDER BY month
            """, (row["node_id"],))

            results.append(ObsessionEntry(
                node_id=row["node_id"],
                surface_form=node["surface_form"],
                node_type=node["node_type"],
                window_mention_count=current_count,
                all_time_mention_count=node["mention_count"],
                trend=trend,
                first_seen=node["first_seen"],
                last_seen=node["last_seen"],
                time_series=[dict(r) for r in time_series],
            ))

        return results

    def get_forgotten_ideas(self, dormant_days: int = 90,
                            min_centrality: float = 0.01) -> list[ObsessionEntry]:
        """
        Forgotten ideas: nodes that were once highly referenced
        but have gone dormant (no mentions in `dormant_days`).
        """
        cutoff = datetime.utcnow() - timedelta(days=dormant_days)
        return self.db.query("""
            SELECT n.node_id, n.surface_form, n.node_type,
                   n.mention_count, n.first_seen, n.last_seen,
                   n.centrality_score
            FROM nodes n
            WHERE n.last_seen < ?
              AND n.centrality_score >= ?
              AND n.mention_count >= 5
            ORDER BY n.centrality_score DESC
            LIMIT 30
        """, (cutoff, min_centrality))

4.7 Vector Search Layer

Purpose

The Vector Search Layer provides sub-second semantic retrieval: given a query or a node's embedding, find the most semantically similar chunks, nodes, or documents across the entire corpus.

Python

# search/vector_index.py

import numpy as np
from abc import ABC, abstractmethod

class VectorIndex(ABC):

    @abstractmethod
    def add(self, id: str, vector: np.ndarray, metadata: dict): ...

    @abstractmethod
    def query(self, vector: np.ndarray, k: int) -> list[tuple[str, float]]: ...

    @abstractmethod
    def delete(self, id: str): ...


class SqliteVecIndex(VectorIndex):
    """
    Uses sqlite-vec (successor to sqlite-vss) for local
    vector similarity search backed by SQLite.
    """

    def __init__(self, db_path: str, dimensions: int = 384):
        import sqlite_vec
        import sqlite3
        self.conn = sqlite3.connect(db_path)
        self.conn.enable_load_extension(True)
        sqlite_vec.load(self.conn)
        self.dimensions = dimensions
        self.conn.execute(f"""
            CREATE VIRTUAL TABLE IF NOT EXISTS vec_chunks
            USING vec0(embedding float[{dimensions}])
        """)
        self.conn.commit()

    def add(self, id: str, vector: np.ndarray, metadata: dict = None):
        self.conn.execute(
            "INSERT OR REPLACE INTO vec_chunks(rowid, embedding) VALUES (?, ?)",
            (self._id_to_rowid(id), vector.tobytes())
        )
        self.conn.commit()

    def query(self, vector: np.ndarray, k: int = 10) -> list[tuple[str, float]]:
        results = self.conn.execute("""
            SELECT rowid, distance
            FROM vec_chunks
            WHERE embedding MATCH ?
            ORDER BY distance
            LIMIT ?
        """, (vector.tobytes(), k)).fetchall()
        return [(self._rowid_to_id(row[0]), 1.0 - row[1]) for row in results]

    def _id_to_rowid(self, id: str) -> int:
        return int(__import__('hashlib').md5(id.encode()).hexdigest()[:8], 16)

    def _rowid_to_id(self, rowid: int) -> str:
        return self.db.get_chunk_id_by_rowid(rowid)  # Lookup table in SQLite


class ChromaIndex(VectorIndex):
    """
    Alternative backend using ChromaDB for richer metadata filtering.
    """

    def __init__(self, persist_dir: str, collection_name: str = "chronomap"):
        import chromadb
        self.client = chromadb.PersistentClient(path=persist_dir)
        self.collection = self.client.get_or_create_collection(
            name=collection_name,
            metadata={"hnsw:space": "cosine"},
        )

    def add(self, id: str, vector: np.ndarray, metadata: dict = None):
        self.collection.upsert(
            ids=[id],
            embeddings=[vector.tolist()],
            metadatas=[metadata or {}],
        )

    def query(self, vector: np.ndarray, k: int = 10) -> list[tuple[str, float]]:
        results = self.collection.query(
            query_embeddings=[vector.tolist()],
            n_results=k,
        )
        ids = results["ids"][0]
        distances = results["distances"][0]
        return [(id_, 1.0 - dist) for id_, dist in zip(ids, distances)]

    def delete(self, id: str):
        self.collection.delete(ids=[id])

4.8 REST API Server (FastAPI)

Purpose

The FastAPI server exposes all graph, search, temporal, and insight data to the Svelte frontend. All endpoints serve only localhost.

Python

# api/server.py

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from api.routers import graph, search, temporal, insights, documents, settings

app = FastAPI(title="ChronoMap API", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],   # Svelte dev server
    allow_methods=["GET", "POST", "DELETE"],
    allow_headers=["*"],
)

app.include_router(graph.router,     prefix="/api/v1/graph")
app.include_router(search.router,    prefix="/api/v1/search")
app.include_router(temporal.router,  prefix="/api/v1/temporal")
app.include_router(insights.router,  prefix="/api/v1/insights")
app.include_router(documents.router, prefix="/api/v1/documents")
app.include_router(settings.router,  prefix="/api/v1/settings")

4.9 Frontend Graph UI (Svelte)

Purpose

The Svelte frontend provides the primary user interface: an interactive, force-directed 3D knowledge graph, a timeline slider for time-travel, a semantic search bar, node detail panels, and insight panels.

Component Structure

text

App.svelte
├── GraphCanvas.svelte          ← D3 force layout + Three.js rendering
│   ├── NodeMesh.svelte         ← Individual node sphere (colored by topic)
│   ├── EdgeLine.svelte         ← Weighted edge line
│   └── ClusterHull.svelte      ← Convex hull around topic clusters
│
├── TimeSlider.svelte           ← Scrub timeline → calls /temporal/graph_at
├── SearchBar.svelte            ← Semantic search → calls /search/semantic
├── NodeDetail.svelte           ← Click node → show sources, relations, timeline
├── BlindSpotPanel.svelte       ← Insight: blind spots with "Connect?" prompt
├── IslandPanel.svelte          ← Insight: isolated clusters
├── ObsessionTracker.svelte     ← Insight: rising/falling concepts chart
├── ForgottenIdeas.svelte       ← Insight: dormant high-centrality concepts
├── ClusterView.svelte          ← Zoomed-out topic cluster summary
└── SettingsPanel.svelte        ← Watch paths, source toggles, model config

UI Sketch

text

┌────────────────────────────────────────────────────────────────────────┐
│  🧠 CHRONOMAP                [Search your knowledge...]   [⚙ Settings] │
├─────────────────────────────────────────┬──────────────────────────────┤
│                                         │  NODE DETAIL                 │
│                                         │  ━━━━━━━━━━━━━━━━━━━━━━━━   │
│         [3D Force Graph Canvas]         │  "neural network"            │
│                                         │  Type: CONCEPT               │
│    ◉ machine learning                   │  First seen: Jan 12, 2022    │
│       │                                 │  Last seen:  May 3, 2024     │
│    ◎ neural network ─────── ◉ backprop  │  Mentions: 47                │
│       │                                 │  ────────────────────────    │
│       ├── ◎ transformer                 │  SOURCES (top 5)             │
│       │                                 │  • deep_learning_notes.md    │
│    ◎ attention                          │  • stanford_cs229.pdf (p.12) │
│                                         │  • github.com/karpathy/nn    │
│  ⚠ BLIND SPOT DETECTED                  │  ────────────────────────    │
│  "privacy" ←──?──► "surveillance"       │  RELATIONS                   │
│  Similarity: 0.89 | Distance: ∞         │  → trained on (data)         │
│  [Connect?]                             │  → used in (GPT-4)           │
│                                         │  ← type of (deep learning)   │
├─────────────────────────────────────────┴──────────────────────────────┤
│  ◄────────────────●────────────────────────────────────────────────►   │
│  Jan 2020                            Now                  [Live]        │
└────────────────────────────────────────────────────────────────────────┘

5. Data Models & Schemas
5.1 SQLite Schema

SQL

-- migrations/001_initial.sql

CREATE TABLE IF NOT EXISTS documents (
    doc_id          TEXT PRIMARY KEY,
    source_type     TEXT NOT NULL,          -- markdown|pdf|browser_bookmark|email|code|ebook|calendar
    uri             TEXT NOT NULL,          -- File path or URL
    title           TEXT,
    author          TEXT,
    created_at      DATETIME NOT NULL,
    modified_at     DATETIME NOT NULL,
    ingested_at     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    content_hash    TEXT NOT NULL,          -- SHA-256 of file content (for change detection)
    chunk_count     INTEGER NOT NULL DEFAULT 0,
    word_count      INTEGER,
    tags            TEXT,                   -- JSON array
    raw_metadata    TEXT                    -- JSON blob of source-specific metadata
);

CREATE TABLE IF NOT EXISTS chunks (
    chunk_id        TEXT PRIMARY KEY,
    doc_id          TEXT NOT NULL,
    text            TEXT NOT NULL,
    chunk_index     INTEGER NOT NULL,
    char_start      INTEGER,
    char_end        INTEGER,
    page_number     INTEGER,
    line_number     INTEGER,
    chunk_type      TEXT DEFAULT 'body',    -- body|highlight|comment|docstring
    metadata        TEXT,                   -- JSON
    FOREIGN KEY (doc_id) REFERENCES documents(doc_id)
);

CREATE TABLE IF NOT EXISTS nodes (
    node_id         TEXT PRIMARY KEY,
    label           TEXT NOT NULL,          -- spaCy NER label or "CONCEPT"
    node_type       TEXT NOT NULL,          -- entity|concept
    surface_form    TEXT NOT NULL,          -- Most common surface form
    canonical_form  TEXT,                   -- Normalized form used for dedup
    first_seen      DATETIME NOT NULL,
    last_seen       DATETIME,
    mention_count   INTEGER NOT NULL DEFAULT 0,
    centrality_score REAL DEFAULT 0.0,      -- Updated periodically (degree centrality)
    topic_id        INTEGER,                -- BERTopic assignment
    topic_label     TEXT,
    embedding       BLOB                    -- Representative embedding (float32 array)
);

CREATE TABLE IF NOT EXISTS edges (
    edge_id         TEXT PRIMARY KEY,
    src             TEXT NOT NULL,
    dst             TEXT NOT NULL,
    edge_type       TEXT NOT NULL,          -- co_mention|relation_verb|explicit_link
    weight          REAL NOT NULL DEFAULT 0.0,
    co_mention_count INTEGER NOT NULL DEFAULT 0,
    first_seen      DATETIME NOT NULL,
    last_seen       DATETIME,
    doc_ids         TEXT,                   -- JSON array of contributing doc IDs
    FOREIGN KEY (src) REFERENCES nodes(node_id),
    FOREIGN KEY (dst) REFERENCES nodes(node_id),
    UNIQUE(src, dst, edge_type)
);

CREATE TABLE IF NOT EXISTS mentions (
    mention_id      TEXT PRIMARY KEY,
    node_id         TEXT NOT NULL,
    doc_id          TEXT NOT NULL,
    chunk_id        TEXT NOT NULL,
    timestamp       DATETIME NOT NULL,      -- Document created_at (not ingestion time)
    surface_form    TEXT,
    confidence      REAL DEFAULT 1.0,
    FOREIGN KEY (node_id) REFERENCES nodes(node_id),
    FOREIGN KEY (doc_id) REFERENCES documents(doc_id),
    FOREIGN KEY (chunk_id) REFERENCES chunks(chunk_id)
);

CREATE TABLE IF NOT EXISTS topics (
    topic_id        INTEGER PRIMARY KEY,
    label           TEXT NOT NULL,          -- BERTopic auto-label
    description     TEXT,                   -- Top representative words
    node_count      INTEGER DEFAULT 0,
    color_hex       TEXT DEFAULT '#888888', -- UI rendering color
    last_updated    DATETIME
);

CREATE TABLE IF NOT EXISTS relation_triples (
    triple_id       TEXT PRIMARY KEY,
    doc_id          TEXT NOT NULL,
    chunk_id        TEXT NOT NULL,
    subject_node_id TEXT,
    predicate       TEXT NOT NULL,
    object_node_id  TEXT,
    subject_surface TEXT,
    object_surface  TEXT,
    confidence      REAL DEFAULT 0.6,
    timestamp       DATETIME NOT NULL,
    FOREIGN KEY (doc_id) REFERENCES documents(doc_id)
);

CREATE TABLE IF NOT EXISTS insight_cache (
    insight_id      TEXT PRIMARY KEY,
    insight_type    TEXT NOT NULL,          -- blind_spot|island|obsession|forgotten
    payload         TEXT NOT NULL,          -- JSON serialized insight
    computed_at     DATETIME NOT NULL,
    expires_at      DATETIME                -- Cache invalidated after re-ingestion
);

CREATE TABLE IF NOT EXISTS settings (
    key             TEXT PRIMARY KEY,
    value           TEXT NOT NULL,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_chunks_doc_id ON chunks(doc_id);
CREATE INDEX IF NOT EXISTS idx_mentions_node_id ON mentions(node_id);
CREATE INDEX IF NOT EXISTS idx_mentions_doc_id ON mentions(doc_id);
CREATE INDEX IF NOT EXISTS idx_mentions_timestamp ON mentions(timestamp);
CREATE INDEX IF NOT EXISTS idx_edges_src ON edges(src);
CREATE INDEX IF NOT EXISTS idx_edges_dst ON edges(dst);
CREATE INDEX IF NOT EXISTS idx_edges_weight ON edges(weight DESC);
CREATE INDEX IF NOT EXISTS idx_nodes_topic_id ON nodes(topic_id);
CREATE INDEX IF NOT EXISTS idx_nodes_last_seen ON nodes(last_seen);
CREATE INDEX IF NOT EXISTS idx_nodes_centrality ON nodes(centrality_score DESC);

6. API Specifications
6.1 Graph Endpoints

text

GET  /api/v1/graph
     Query: ?limit_nodes=500&min_edge_weight=0.1&topic_filter=&node_type=
     Returns: {
       nodes: GraphNode[],
       edges: GraphEdge[],
       stats: { node_count, edge_count, topic_count, doc_count }
     }

GET  /api/v1/graph/node/:node_id
     Returns: {
       node: GraphNode,
       neighbors: GraphNode[],
       edges: GraphEdge[],
       sources: Document[],
       timeline: { month, count }[],
       topic: Topic
     }

GET  /api/v1/graph/subgraph/:node_id
     Query: ?depth=2&min_weight=0.05
     Returns: { nodes: GraphNode[], edges: GraphEdge[] }

GET  /api/v1/graph/topics
     Returns: Topic[]

GET  /api/v1/graph/stats
     Returns: {
       total_nodes, total_edges, total_docs, total_chunks,
       top_nodes_by_centrality: GraphNode[],
       nodes_by_type: { entity, concept },
       ingestion_rate_last_7d: { date, count }[]
     }

6.2 Search Endpoints

text

POST /api/v1/search/semantic
     Body: { query: string, k: int = 10, filters: { source_type?, date_from?, date_to? } }
     Returns: {
       results: [{
         chunk_id, doc_id, text, score,
         document: { title, uri, source_type },
         related_nodes: GraphNode[]
       }]
     }

POST /api/v1/search/keyword
     Body: { query: string, limit: int = 20 }
     Returns: { results: Document[] }

POST /api/v1/search/node
     Body: { query: string, k: int = 10 }
     Returns: { results: GraphNode[] }

6.3 Temporal Endpoints

text

GET  /api/v1/temporal/graph_at
     Query: ?timestamp=2023-06-01T00:00:00Z&limit_nodes=500
     Returns: { nodes: GraphNode[], edges: GraphEdge[], timestamp }

GET  /api/v1/temporal/diff
     Query: ?t1=2023-01-01T00:00:00Z&t2=2024-01-01T00:00:00Z
     Returns: {
       nodes_added: GraphNode[],
       nodes_removed: GraphNode[],
       edges_added: GraphEdge[],
       edges_removed: GraphEdge[]
     }

GET  /api/v1/temporal/node/:node_id/timeline
     Returns: { month, mention_count }[]

GET  /api/v1/temporal/snapshots
     Returns: list of suggested meaningful snapshots (first doc ingested,
              6-month intervals, last year boundary, etc.)

6.4 Insight Endpoints

text

GET  /api/v1/insights/blind_spots
     Query: ?max_results=20&similarity_threshold=0.75&min_graph_distance=4
     Returns: BlindSpot[]

GET  /api/v1/insights/islands
     Returns: KnowledgeIsland[]

GET  /api/v1/insights/obsessions
     Query: ?window_days=30&top_n=20
     Returns: ObsessionEntry[]

GET  /api/v1/insights/forgotten
     Query: ?dormant_days=90&min_centrality=0.01
     Returns: ObsessionEntry[]

POST /api/v1/insights/refresh
     Triggers async re-computation of all insights
     Returns: { job_id, estimated_seconds }

6.5 Document Endpoints

text

GET  /api/v1/documents
     Query: ?source_type=&limit=50&offset=0&from=&to=
     Returns: { total, items: Document[] }

GET  /api/v1/documents/:doc_id
     Returns: Document (with chunks, extracted nodes)

DELETE /api/v1/documents/:doc_id
     Removes document and all associated mentions/edges
     Returns: { success, nodes_affected, edges_recalculated }

POST /api/v1/documents/reindex
     Body: { doc_ids: string[] }  or  { all: true }
     Returns: { job_id }

6.6 Settings Endpoints

text

GET  /api/v1/settings
     Returns: ChronoMapSettings

PUT  /api/v1/settings
     Body: Partial<ChronoMapSettings>
     Returns: ChronoMapSettings

GET  /api/v1/settings/watch_paths
     Returns: string[]

POST /api/v1/settings/watch_paths
     Body: { path: string }
     Returns: { success, path }

DELETE /api/v1/settings/watch_paths
     Body: { path: string }
     Returns: { success }

GET  /api/v1/health
     Returns: {
       daemon_running, api_healthy, db_healthy,
       vector_index_healthy, node_count, edge_count, version
     }

7. Directory Structure

text

chronomap/
├── chronomap/                          # Main Python package
│   │
│   ├── ingestors/
│   │   ├── __init__.py
│   │   ├── base.py                     # BaseIngestor, Document, TextChunk
│   │   ├── registry.py                 # IngestorRegistry (extension → ingestor dispatch)
│   │   ├── markdown_ingestor.py        # .md, .mdx
│   │   ├── pdf_ingestor.py             # .pdf (PyMuPDF, highlights)
│   │   ├── browser_ingestor.py         # bookmarks.html, places.sqlite, History
│   │   ├── email_ingestor.py           # .mbox, .eml
│   │   ├── code_ingestor.py            # .py/.ts/.go/.rs (docstrings + comments)
│   │   ├── ebook_ingestor.py           # Kindle clippings, Kobo SQLite
│   │   └── calendar_ingestor.py        # .ics
│   │
│   ├── extractor/
│   │   ├── __init__.py
│   │   ├── pipeline.py                 # ExtractionPipeline: orchestrates all extractors
│   │   ├── entity_extractor.py         # spaCy NER → EntityMention
│   │   ├── concept_extractor.py        # KeyBERT → ConceptMention
│   │   ├── embedder.py                 # SentenceTransformers → ChunkEmbedding
│   │   ├── relation_extractor.py       # Dependency parse → RelationTriple
│   │   └── topic_modeler.py            # BERTopic → TopicAssignment
│   │
│   ├── graph/
│   │   ├── __init__.py
│   │   ├── builder.py                  # GraphBuilder: upserts nodes/edges
│   │   ├── merger.py                   # NodeMerger: entity deduplication
│   │   ├── scorer.py                   # EdgeScorer: co-mention + semantic weight
│   │   ├── temporal.py                 # TemporalGraph: graph_at(t), diff, timeline
│   │   ├── blind_spot_detector.py      # BlindSpotDetector
│   │   ├── island_detector.py          # IslandDetector (Louvain communities)
│   │   └── obsession_tracker.py        # ObsessionTracker + ForgottenIdeas
│   │
│   ├── search/
│   │   ├── __init__.py
│   │   ├── vector_index.py             # VectorIndex ABC + SqliteVecIndex + ChromaIndex
│   │   └── search_engine.py            # SemanticSearch, KeywordSearch orchestration
│   │
│   ├── daemon/
│   │   ├── __init__.py
│   │   └── watch_daemon.py             # ChronoMapDaemon (watchdog-based)
│   │
│   ├── storage/
│   │   ├── __init__.py
│   │   ├── db.py                       # GraphDB: SQLite connection + query helpers
│   │   ├── migrations.py               # Migration runner
│   │   ├── document_repo.py            # Document CRUD
│   │   ├── chunk_repo.py               # Chunk CRUD
│   │   ├── node_repo.py                # Node CRUD + embedding store/retrieve
│   │   ├── edge_repo.py                # Edge CRUD
│   │   ├── mention_repo.py             # Mention CRUD + time-series queries
│   │   ├── insight_cache_repo.py       # Insight cache CRUD
│   │   └── settings_repo.py            # Settings key/value store
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── server.py                   # FastAPI app factory
│   │   ├── middleware.py               # LocalOnly enforcement, CORS, logging
│   │   ├── routers/
│   │   │   ├── graph.py                # /api/v1/graph routes
│   │   │   ├── search.py               # /api/v1/search routes
│   │   │   ├── temporal.py             # /api/v1/temporal routes
│   │   │   ├── insights.py             # /api/v1/insights routes
│   │   │   ├── documents.py            # /api/v1/documents routes
│   │   │   └── settings.py             # /api/v1/settings + health routes
│   │   └── schemas/
│   │       ├── graph.py                # Pydantic schemas: GraphNode, GraphEdge, Topic
│   │       ├── search.py               # SearchRequest, SearchResult
│   │       ├── temporal.py             # TemporalRequest, GraphDiff
│   │       ├── insights.py             # BlindSpot, Island, Obsession
│   │       └── settings.py             # ChronoMapSettings
│   │
│   └── config/
│       ├── __init__.py
│       ├── config.py                   # Config dataclass + TOML loader
│       └── defaults.py                 # Default configuration values
│
├── ui/                                 # Svelte frontend
│   ├── src/
│   │   ├── App.svelte
│   │   ├── pages/
│   │   │   └── Main.svelte
│   │   ├── components/
│   │   │   ├── GraphCanvas.svelte      # D3 force + Three.js renderer
│   │   │   ├── TimeSlider.svelte       # Timeline scrubber
│   │   │   ├── SearchBar.svelte        # Semantic search input
│   │   │   ├── NodeDetail.svelte       # Node info panel
│   │   │   ├── BlindSpotPanel.svelte   # Blind spot insight panel
│   │   │   ├── IslandPanel.svelte      # Island insight panel
│   │   │   ├── ObsessionTracker.svelte # Obsession/trend chart
│   │   │   ├── ForgottenIdeas.svelte   # Forgotten ideas panel
│   │   │   ├── ClusterView.svelte      # Topic cluster zoom-out view
│   │   │   └── SettingsPanel.svelte    # Watch paths, toggles, model config
│   │   ├── stores/
│   │   │   ├── graph.ts                # Svelte writable: graph data
│   │   │   ├── temporal.ts             # Svelte writable: current timestamp
│   │   │   ├── insights.ts             # Svelte writable: all insights
│   │   │   └── ui.ts                   # Svelte writable: selected node, zoom, mode
│   │   ├── api/
│   │   │   └── client.ts               # Typed API client (fetch-based)
│   │   └── types/
│   │       └── index.ts                # GraphNode, GraphEdge, BlindSpot, etc.
│   ├── package.json
│   ├── vite.config.ts
│   ├── svelte.config.js
│   └── tsconfig.json
│
├── tests/
│   ├── unit/
│   │   ├── test_ingestors.py
│   │   ├── test_entity_extractor.py
│   │   ├── test_concept_extractor.py
│   │   ├── test_merger.py
│   │   ├── test_scorer.py
│   │   ├── test_temporal.py
│   │   ├── test_blind_spot_detector.py
│   │   └── test_obsession_tracker.py
│   ├── integration/
│   │   ├── test_pipeline_end_to_end.py
│   │   ├── test_api_graph.py
│   │   ├── test_api_search.py
│   │   └── test_api_temporal.py
│   └── fixtures/
│       ├── sample_notes/
│       ├── sample.pdf
│       └── sample_bookmarks.html
│
├── data/                               # Runtime data (gitignored)
│   ├── chronomap.db
│   └── vector_index/
│
├── logs/                               # Opt-in logs (gitignored)
│
├── pyproject.toml
├── requirements.txt
├── requirements-dev.txt
├── Makefile
├── Dockerfile
├── docker-compose.yml
└── README.md

8. Configuration System
8.1 Config File (TOML)

Stored at ~/.chronomap/config.toml

toml

[daemon]
enabled = true
debounce_ms = 100
watch_paths = [
    "~/Documents/Notes",
    "~/Documents/Papers",
    "~/Downloads",
]
exclude_patterns = [
    ".git",
    "node_modules",
    "__pycache__",
    ".DS_Store",
    "*.tmp",
]

[ingestors]
enabled = [
    "markdown",
    "pdf",
    "browser_bookmark",
    "email",
    "code",
    "ebook",
    "calendar",
]

[ingestors.pdf]
extract_highlights = true
extract_body = true
min_text_length = 50        # Skip pages with less than 50 chars

[ingestors.browser]
history_db_path = ""        # Auto-detected if empty
bookmarks_path = ""
include_visited_pages = true
min_visit_duration_seconds = 5

[ingestors.email]
mbox_paths = []
include_sent = false
exclude_domains = ["noreply@", "no-reply@"]

[ingestors.code]
languages = ["python", "typescript", "go", "rust", "javascript"]
extract_comments = true
extract_docstrings = true
extract_identifiers = false  # Variable/function names — too noisy by default

[extraction]
spacy_model = "en_core_web_trf"     # or "en_core_web_sm" for lower RAM
embedding_model = "sentence-transformers/all-MiniLM-L6-v2"
embedding_batch_size = 64
chunk_size_words = 512
chunk_overlap_words = 64
top_concepts_per_chunk = 8
min_concept_score = 0.20
entity_labels = ["PERSON", "ORG", "GPE", "PRODUCT", "EVENT", "WORK_OF_ART", "LAW", "NORP"]
enable_relation_extraction = true
enable_topic_modeling = true
topic_model_min_size = 10

[graph]
merge_similarity_threshold = 0.92
edge_weight_alpha = 0.6         # Co-mention frequency weight
edge_weight_beta = 0.40         # Semantic similarity weight
centrality_update_interval_hours = 6

[vector_index]
backend = "sqlite-vec"          # "sqlite-vec" | "chroma"
dimensions = 384                # Must match embedding model output
chroma_persist_dir = "~/.chronomap/vector_index"

[insights]
blind_spot_similarity_threshold = 0.75
blind_spot_min_graph_distance = 4
blind_spot_top_k_neighbors = 20
obsession_window_days = 30
obsession_top_n = 20
forgotten_dormant_days = 90
forgotten_min_centrality = 0.01
auto_refresh_interval_hours = 12

[api]
host = "127.0.0.1"
port = 7331
workers = 1

[storage]
db_path = "~/.chronomap/chronomap.db"
log_retention_days = 365
max_db_size_mb = 2000

[ui]
port = 5173
open_browser_on_start = true
default_node_limit = 500
default_min_edge_weight = 0.05
graph_3d = false            # True = Three.js, False = D3 only (lower GPU)

[logging]
level = "info"
file = "~/.chronomap/logs/chronomap.log"
max_size_mb = 50
max_backups = 5

8.2 Config Loader

Python

# config/config.py

import tomllib
import os
from pathlib import Path
from dataclasses import dataclass, field

@dataclass
class ChronoMapConfig:
    daemon: DaemonConfig
    ingestors: IngestorConfig
    extraction: ExtractionConfig
    graph: GraphConfig
    vector_index: VectorIndexConfig
    insights: InsightConfig
    api: APIConfig
    storage: StorageConfig
    ui: UIConfig
    logging: LoggingConfig

def load(path: str | None = None) -> ChronoMapConfig:
    config_path = path or os.path.expanduser("~/.chronomap/config.toml")
    p = Path(config_path)

    if not p.exists():
        os.makedirs(p.parent, exist_ok=True)
        defaults = _defaults()
        _save(defaults, config_path)
        return defaults

    with open(config_path, "rb") as f:
        raw = tomllib.load(f)

    return _merge_with_defaults(raw)

def default_path() -> str:
    return os.path.expanduser("~/.chronomap/config.toml")

9. Ingestion Source Registry

Python

# ingestors/registry.py

from pathlib import Path

class IngestorRegistry:

    def __init__(self, config):
        self._ingestors = []
        self._register_enabled(config)

    def _register_enabled(self, config):
        enabled = set(config.ingestors.enabled)

        if "markdown" in enabled:
            from .markdown_ingestor import MarkdownIngestor
            self._ingestors.append(MarkdownIngestor())

        if "pdf" in enabled:
            from .pdf_ingestor import PDFIngestor
            self._ingestors.append(PDFIngestor(config.ingestors.pdf))

        if "browser_bookmark" in enabled:
            from .browser_ingestor import BrowserIngestor
            self._ingestors.append(BrowserIngestor(config.ingestors.browser))

        if "email" in enabled:
            from .email_ingestor import EmailIngestor
            self._ingestors.append(EmailIngestor(config.ingestors.email))

        if "code" in enabled:
            from .code_ingestor import CodeIngestor
            self._ingestors.append(CodeIngestor(config.ingestors.code))

        if "ebook" in enabled:
            from .ebook_ingestor import EbookIngestor
            self._ingestors.append(EbookIngestor())

        if "calendar" in enabled:
            from .calendar_ingestor import CalendarIngestor
            self._ingestors.append(CalendarIngestor())

    def find(self, path: str):
        for ingestor in self._ingestors:
            if ingestor.can_handle(path):
                return ingestor
        return None

    def find_all(self, path: str):
        """Some files may match multiple ingestors (e.g. .md in code repo)."""
        return [i for i in self._ingestors if i.can_handle(path)]

10. NLP Pipeline Deep Dive
10.1 Pipeline Orchestration

Python

# extractor/pipeline.py

import logging
from concurrent.futures import ThreadPoolExecutor

log = logging.getLogger(__name__)

class ExtractionPipeline:
    """
    Orchestrates all extractors per document.
    Runs embedding in batch (efficient GPU use).
    Runs NER/concept/relation per chunk (parallelized).
    BERTopic runs asynchronously in batch after ingestion.
    """

    def __init__(self, config, db, graph_builder, vector_index):
        self.config = config
        self.db = db
        self.graph_builder = graph_builder
        self.vector_index = vector_index

        self.entity_extractor = EntityExtractor(config.extraction.spacy_model)
        self.concept_extractor = ConceptExtractor(config.extraction.embedding_model)
        self.embedder = Embedder(config.extraction.embedding_model,
                                 config.extraction.embedding_batch_size)
        self.relation_extractor = RelationExtractor(self.entity_extractor.nlp)
        self.topic_modeler = TopicModeler(config.extraction.topic_model_min_size)

        self._pending_for_topic_model: list = []  # Batched for async BERTopic

    def process(self, document) -> None:
        try:
            # 1. Persist document and chunks
            self.db.upsert_document(document)
            for chunk in document.chunks:
                self.db.upsert_chunk(chunk)

            # 2. Batch-embed all chunks
            embeddings = self.embedder.embed_chunks(document.chunks)
            for emb in embeddings:
                self.db.upsert_chunk_embedding(emb.chunk_id, emb.vector)
                self.vector_index.add(emb.chunk_id, emb.vector, {
                    "doc_id": emb.doc_id,
                    "source_type": document.source_type,
                })

            # 3. Per-chunk extraction (parallelized)
            all_entities, all_concepts, all_relations = [], [], []
            with ThreadPoolExecutor(max_workers=4) as executor:
                futures = {
                    executor.submit(self._extract_chunk, chunk): chunk
                    for chunk in document.chunks
                }
                for future in futures:
                    entities, concepts, relations = future.result()
                    all_entities.extend(entities)
                    all_concepts.extend(concepts)
                    all_relations.extend(relations)

            # 4. Update graph
            self.graph_builder.process_extractions(
                doc_id=document.doc_id,
                entities=all_entities,
                concepts=all_concepts,
                triples=all_relations,
                timestamp=document.modified_at,
            )

            # 5. Queue for BERTopic batch (run periodically, not per-doc)
            self._pending_for_topic_model.extend(zip(document.chunks, embeddings))
            if len(self._pending_for_topic_model) >= 500:
                self._run_topic_modeling()

        except Exception as e:
            # Never let extraction errors cascade — log and continue
            log.error(f"Extraction failed for {document.doc_id}: {e}", exc_info=True)

    def _extract_chunk(self, chunk):
        entities = self.entity_extractor.extract(chunk)
        concepts = self.concept_extractor.extract(chunk,
                    top_n=self.config.extraction.top_concepts_per_chunk)
        relations = self.relation_extractor.extract(chunk) \
                    if self.config.extraction.enable_relation_extraction else []
        return entities, concepts, relations

    def _run_topic_modeling(self):
        if not self._pending_for_topic_model:
            return
        chunks, embeddings = zip(*self._pending_for_topic_model)
        import numpy as np
        emb_matrix = np.array([e.vector for e in embeddings])
        assignments = self.topic_modeler.transform(list(chunks), emb_matrix)
        for assignment in assignments:
            self.db.update_chunk_topic(assignment.chunk_id, assignment.topic_id,
                                       assignment.topic_label, assignment.probability)
        self._pending_for_topic_model.clear()

    def handle_source_deletion(self, path: str):
        """Mark all nodes/edges from a deleted file as last_seen = now."""
        doc = self.db.get_document_by_uri(path)
        if doc:
            self.db.mark_document_deleted(doc["doc_id"])
            self.db.update_nodes_last_seen_by_doc(doc["doc_id"])

10.2 Model Loading Strategy

Python

# extractor/model_manager.py

"""
All models are loaded once at startup and reused across all ingestion cycles.
Models are NOT reloaded per-document — this is critical for performance.
"""

class ModelManager:

    _instance = None

    @classmethod
    def get(cls, config) -> "ModelManager":
        if cls._instance is None:
            cls._instance = cls(config)
        return cls._instance

    def __init__(self, config):
        import spacy
        from sentence_transformers import SentenceTransformer
        from keybert import KeyBERT
        from bertopic import BERTopic

        log.info("Loading NLP models (first-time load may take 30-60s)...")

        self.spacy_nlp = spacy.load(config.extraction.spacy_model)
        log.info(f"spaCy loaded: {config.extraction.spacy_model}")

        self.sentence_model = SentenceTransformer(config.extraction.embedding_model)
        log.info(f"SentenceTransformer loaded: {config.extraction.embedding_model}")

        self.keybert_model = KeyBERT(model=self.sentence_model)
        log.info("KeyBERT loaded (reusing SentenceTransformer)")

        self.bertopic_model = BERTopic(
            embedding_model=self.sentence_model,  # Reuse embedder
            min_topic_size=config.extraction.topic_model_min_size,
            calculate_probabilities=True,
            verbose=False,
        )
        log.info("BERTopic loaded")

11. Graph Model Deep Dive
11.1 Node Types and Labels
Node Type	Labels	Example Surface Forms
entity	PERSON	"Alan Turing", "Andrej Karpathy"
entity	ORG	"OpenAI", "MIT", "DeepMind"
entity	GPE	"San Francisco", "Germany"
entity	PRODUCT	"GPT-4", "PyTorch"
entity	WORK_OF_ART	"Attention is All You Need"
entity	EVENT	"NeurIPS 2023"
concept	CONCEPT	"attention mechanism", "gradient descent", "privacy by design"
11.2 Edge Types
Edge Type	Source	Weight Basis
co_mention	Two nodes appear in the same chunk	Frequency × semantic similarity
relation_verb	Extracted SVO triple (e.g., "learns from")	Relation extraction confidence
topic_cluster	Two nodes share the same BERTopic topic	Topic probability product
11.3 Node Centrality Update

Centrality is recomputed on a schedule (not on every ingestion) because it requires loading the full graph:

Python

# graph/centrality_updater.py

import networkx as nx

class CentralityUpdater:

    def update(self, G: nx.Graph, db):
        """Recompute degree centrality and update nodes table."""
        centrality = nx.degree_centrality(G)
        # Optionally also: betweenness (expensive), pagerank
        pagerank = nx.pagerank(G, weight="weight")

        batch = [
            (centrality[n], pagerank[n], n)
            for n in G.nodes()
        ]
        db.batch_update_node_centrality(batch)

12. Temporal Engine Deep Dive
12.1 Temporal Semantics
Event	How It's Recorded
Entity first seen	nodes.first_seen = document modified_at (not ingestion time)
Entity last seen	nodes.last_seen updated on each mention; NULL = still active
Edge first seen	edges.first_seen = first co-mention timestamp
Edge last seen	edges.last_seen updated on each co-mention; NULL = still active
Document deleted	documents.deleted_at set; all mentions trigger nodes.last_seen update
12.2 Snapshot Suggestions

Python

# graph/temporal.py (continued)

def suggest_snapshots(self) -> list[dict]:
    """
    Return meaningful snapshot timestamps to surface in the UI time slider:
    - First document ingested
    - Year boundaries
    - Six-month boundaries
    - Most recent large ingestion batch
    """
    first_doc = self.db.query_one(
        "SELECT MIN(created_at) as ts FROM documents"
    )
    last_doc = self.db.query_one(
        "SELECT MAX(modified_at) as ts FROM documents"
    )

    snapshots = []
    if first_doc and first_doc["ts"]:
        snapshots.append({"label": "First document", "timestamp": first_doc["ts"]})

    # Year boundaries between first and last
    start_year = datetime.fromisoformat(first_doc["ts"]).year
    end_year = datetime.fromisoformat(last_doc["ts"]).year
    for year in range(start_year, end_year + 1):
        snapshots.append({
            "label": f"Jan {year}",
            "timestamp": datetime(year, 1, 1).isoformat()
        })

    snapshots.append({"label": "Now", "timestamp": datetime.utcnow().isoformat()})
    return snapshots

13. Insight Engine Deep Dive
13.1 Insight Caching

Insights are expensive to compute (kNN + path queries). They are cached with a TTL and invalidated after large ingestion batches:

Python

# graph/insight_engine.py

import json
from datetime import datetime, timedelta

class InsightEngine:

    CACHE_TTL_HOURS = 12

    def __init__(self, db, vector_index, G, temporal):
        self.db = db
        self.vector_index = vector_index
        self.G = G
        self.temporal = temporal
        self.blind_spot_detector = BlindSpotDetector(db, vector_index, G)
        self.island_detector = IslandDetector(G, db)
        self.obsession_tracker = ObsessionTracker(db)

    def get_blind_spots(self, force_refresh: bool = False) -> list:
        cached = self._get_cache("blind_spot", force_refresh)
        if cached:
            return cached
        results = self.blind_spot_detector.detect()
        self._set_cache("blind_spot", results)
        return results

    def get_islands(self, force_refresh: bool = False) -> list:
        cached = self._get_cache("island", force_refresh)
        if cached:
            return cached
        results = self.island_detector.detect()
        self._set_cache("island", results)
        return results

    def get_obsessions(self, window_days: int = 30) -> list:
        # Obsessions are time-window-dependent — use shorter TTL (1 hour)
        cache_key = f"obsession_{window_days}d"
        cached = self._get_cache(cache_key, False, ttl_hours=1)
        if cached:
            return cached
        results = self.obsession_tracker.get_obsessions(window_days)
        self._set_cache(cache_key, results, ttl_hours=1)
        return results

    def get_forgotten(self, dormant_days: int = 90) -> list:
        cached = self._get_cache(f"forgotten_{dormant_days}d", False, ttl_hours=6)
        if cached:
            return cached
        results = self.obsession_tracker.get_forgotten_ideas(dormant_days)
        self._set_cache(f"forgotten_{dormant_days}d", results, ttl_hours=6)
        return results

    def _get_cache(self, insight_type: str, force_refresh: bool,
                   ttl_hours: float = None) -> list | None:
        if force_refresh:
            return None
        ttl = ttl_hours or self.CACHE_TTL_HOURS
        row = self.db.query_one("""
            SELECT payload, computed_at FROM insight_cache
            WHERE insight_type = ?
              AND computed_at >= datetime('now', ?)
        """, (insight_type, f"-{ttl} hours"))
        if row:
            return json.loads(row["payload"])
        return None

    def _set_cache(self, insight_type: str, data: list, ttl_hours: float = None):
        import uuid
        ttl = ttl_hours or self.CACHE_TTL_HOURS
        self.db.execute("""
            INSERT OR REPLACE INTO insight_cache (insight_id, insight_type, payload, computed_at, expires_at)
            VALUES (?, ?, ?, CURRENT_TIMESTAMP, datetime('now', ?))
        """, (str(uuid.uuid4()), insight_type, json.dumps([d.__dict__ if hasattr(d, '__dict__') else d for d in data]),
              f"+{ttl} hours"))

    def invalidate_all(self):
        self.db.execute("DELETE FROM insight_cache")

14. Vector Search Deep Dive
14.1 Search Engine

Python

# search/search_engine.py

import numpy as np

class SearchEngine:

    def __init__(self, embedder, vector_index, db):
        self.embedder = embedder
        self.vector_index = vector_index
        self.db = db

    def semantic_search(self, query: str, k: int = 10,
                        filters: dict = None) -> list[dict]:
        """
        1. Embed the query
        2. kNN search in vector index → top-k chunk IDs with scores
        3. Hydrate with chunk text, document metadata, related nodes
        """
        query_vector = self.embedder.embed_query(query)
        raw_results = self.vector_index.query(query_vector, k=k * 2)  # Over-fetch for filter

        results = []
        seen_docs = set()

        for chunk_id, score in raw_results:
            chunk = self.db.get_chunk(chunk_id)
            if chunk is None:
                continue

            doc = self.db.get_document(chunk["doc_id"])
            if doc is None:
                continue

            # Apply filters
            if filters:
                if filters.get("source_type") and doc["source_type"] != filters["source_type"]:
                    continue
                if filters.get("date_from") and doc["created_at"] < filters["date_from"]:
                    continue
                if filters.get("date_to") and doc["created_at"] > filters["date_to"]:
                    continue

            # Deduplicate by document (show best chunk per doc)
            if doc["doc_id"] in seen_docs:
                continue
            seen_docs.add(doc["doc_id"])

            related_nodes = self.db.get_nodes_for_chunk(chunk_id, limit=5)

            results.append({
                "chunk_id": chunk_id,
                "doc_id": chunk["doc_id"],
                "text": chunk["text"],
                "score": float(score),
                "document": dict(doc),
                "related_nodes": [dict(n) for n in related_nodes],
            })

            if len(results) >= k:
                break

        return results

    def node_search(self, query: str, k: int = 10) -> list[dict]:
        """Find nodes most semantically similar to the query."""
        query_vector = self.embedder.embed_query(query)
        # Query node embeddings (stored separately in DB as node-level aggregated embeddings)
        node_results = self.db.query("""
            SELECT node_id, surface_form, node_type, label, mention_count,
                   vec_distance_cosine(embedding, ?) as distance
            FROM nodes
            WHERE embedding IS NOT NULL
            ORDER BY distance ASC
            LIMIT ?
        """, (query_vector.tobytes(), k))
        return [
            {**dict(r), "score": 1.0 - r["distance"]}
            for r in node_results
        ]

15. Privacy & Local-First Model
15.1 Trust Model

text

┌─────────────────────────────────────────┐
│  FULLY TRUSTED (localhost)              │
│  - Watch Daemon process                 │
│  - FastAPI server (127.0.0.1:7331)      │
│  - SQLite database                      │
│  - Vector index                         │
│  - All NLP models (local weights)       │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│  SEMI-TRUSTED (browser)                 │
│  - Svelte UI (localhost:5173)           │
│  - Reads from API only                  │
│  - No direct DB access                  │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│  UNTRUSTED                              │
│  - Everything outside the machine       │
│  - ChronoMap makes ZERO outbound calls  │
└─────────────────────────────────────────┘

15.2 API Local-Only Enforcement

Python

# api/middleware.py

from fastapi import Request
from fastapi.responses import JSONResponse
import ipaddress

async def local_only_middleware(request: Request, call_next):
    client_host = request.client.host if request.client else "unknown"
    try:
        ip = ipaddress.ip_address(client_host)
        if not (ip.is_loopback or ip.is_link_local):
            return JSONResponse(
                status_code=403,
                content={"error": "ChronoMap API is local only"}
            )
    except ValueError:
        return JSONResponse(status_code=403, content={"error": "Invalid client address"})

    return await call_next(request)

15.3 Data Minimization Policy
What is stored	Why	Retention
Document metadata (title, path, dates)	Graph provenance	User-configurable (default: indefinite)
Text chunks	Required for search + re-extraction	Deletable per-document
Embeddings	Vector search	Rebuilt on demand
Entity/concept nodes	Core graph data	Indefinite
Mention timestamps	Temporal features	User-configurable
Raw file content	Never stored	Never
Browser history text	Only page title + URL	Optional, user-controlled
16. Storage & Persistence
16.1 Database Connection

Python

# storage/db.py

import sqlite3
import threading
from pathlib import Path

class GraphDB:

    def __init__(self, db_path: str):
        Path(db_path).parent.mkdir(parents=True, exist_ok=True)
        # WAL mode for concurrent reads during writes
        self._local = threading.local()
        self._db_path = db_path
        self._run_migrations()

    def _conn(self) -> sqlite3.Connection:
        """Thread-local connection — SQLite is not thread-safe for shared connections."""
        if not hasattr(self._local, "conn"):
            conn = sqlite3.connect(
                self._db_path,
                check_same_thread=False,
                timeout=10,
            )
            conn.row_factory = sqlite3.Row
            conn.execute("PRAGMA journal_mode=WAL")
            conn.execute("PRAGMA synchronous=NORMAL")
            conn.execute("PRAGMA foreign_keys=ON")
            conn.execute("PRAGMA cache_size=-64000")    # 64MB cache
            self._local.conn = conn
        return self._local.conn

    def execute(self, sql: str, params: tuple = ()):
        conn = self._conn()
        cursor = conn.execute(sql, params)
        conn.commit()
        return cursor

    def query(self, sql: str, params: tuple = ()) -> list[sqlite3.Row]:
        return self._conn().execute(sql, params).fetchall()

    def query_one(self, sql: str, params: tuple = ()) -> sqlite3.Row | None:
        return self._conn().execute(sql, params).fetchone()

    def _run_migrations(self):
        from .migrations import run_migrations
        run_migrations(self._conn())

16.2 Graph Persistence Strategy
Data	Storage	Access Pattern
Documents, chunks, metadata	SQLite documents + chunks	Write-once, read-often
Nodes, edges	SQLite nodes + edges	Frequent upserts, complex queries
Embeddings (chunk-level)	SQLite-vec virtual table	kNN queries
Embeddings (node-level)	SQLite BLOB in nodes	Dedup + centrality
NetworkX graph	In-memory (rebuilt on startup)	Centrality, path queries
BERTopic model	Pickle file in ~/.chronomap/	Loaded once, reused
Insight cache	SQLite insight_cache	Invalidated on ingestion
17. Logging, Observability & Debugging
17.1 Structured Logging

Python

# config/logging_setup.py

import logging
import logging.handlers
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "ts": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "module": record.module,
            "msg": record.getMessage(),
            "exc": self.formatException(record.exc_info) if record.exc_info else None,
        })

def setup_logging(config):
    root = logging.getLogger("chronomap")
    root.setLevel(getattr(logging, config.logging.level.upper(), logging.INFO))

    handler = logging.handlers.RotatingFileHandler(
        config.logging.file,
        maxBytes=config.logging.max_size_mb * 1024 * 1024,
        backupCount=config.logging.max_backups,
    )
    handler.setFormatter(JSONFormatter())
    root.addHandler(handler)

    # Console handler for interactive use
    console = logging.StreamHandler()
    console.setFormatter(logging.Formatter("[%(levelname)s] %(module)s: %(message)s"))
    root.addHandler(console)

17.2 Ingestion Metrics

The /api/v1/health endpoint returns live ingestion metrics:

Python

# api/routers/settings.py

@router.get("/health")
async def health(db: GraphDB = Depends(get_db)):
    return {
        "api_healthy": True,
        "db_healthy": db.is_healthy(),
        "daemon_running": daemon_state.is_running(),
        "node_count": db.query_one("SELECT COUNT(*) as c FROM nodes")["c"],
        "edge_count": db.query_one("SELECT COUNT(*) as c FROM edges")["c"],
        "doc_count": db.query_one("SELECT COUNT(*) as c FROM documents")["c"],
        "chunk_count": db.query_one("SELECT COUNT(*) as c FROM chunks")["c"],
        "last_ingestion": db.query_one(
            "SELECT MAX(ingested_at) as ts FROM documents"
        )["ts"],
        "pending_topic_modeling": pipeline_state.pending_count,
        "version": "1.0.0",
    }

17.3 Debug Mode

When config.logging.level = "debug":

    Every chunk extraction logged (entity count, concept count, relation count)
    Every graph upsert logged (node_id, merge decision, edge weight delta)
    Every daemon event logged (path, event type, hash match/miss)
    Every kNN query logged (query, k, top result + score)
    Every insight computation logged (input size, output count, duration)

18. Testing Strategy
18.1 Test Pyramid

text

               ┌───────────┐
               │  E2E (5%) │
               │ Full pipeline │
               └─────┬─────┘
        ┌────────────┴────────────┐
        │  Integration (25%)      │
        │  Pipeline + DB + API    │
        └────────────┬────────────┘
   ┌────────────────────────────────────┐
   │         Unit Tests (70%)           │
   │  Ingestors, Extractors, Graph ops  │
   └────────────────────────────────────┘

18.2 Unit Tests

Python

# tests/unit/test_ingestors.py

def test_markdown_ingestor_parses_frontmatter(tmp_path):
    md = tmp_path / "test.md"
    md.write_text("---\ntitle: Test Note\nauthor: Alice\ntags: [ai, research]\n---\n\nContent here.")
    ingestor = MarkdownIngestor()
    docs = ingestor.ingest(str(md))
    assert len(docs) == 1
    assert docs[0].title == "Test Note"
    assert docs[0].author == "Alice"
    assert "ai" in docs[0].tags
    assert len(docs[0].chunks) >= 1

def test_markdown_ingestor_chunks_long_text(tmp_path):
    md = tmp_path / "long.md"
    md.write_text("word " * 2000)
    ingestor = MarkdownIngestor()
    docs = ingestor.ingest(str(md))
    assert len(docs[0].chunks) > 1  # Should be chunked

# tests/unit/test_merger.py

def test_merger_deduplicates_exact_normalized(mock_db, mock_embedder):
    merger = NodeMerger(mock_db, mock_embedder)
    m1 = make_mention("Alan Turing", "PERSON")
    m2 = make_mention("alan turing", "PERSON")
    id1 = merger.resolve_entity(m1)
    id2 = merger.resolve_entity(m2)
    assert id1 == id2

def test_merger_creates_new_node_for_distinct_entity(mock_db, mock_embedder):
    merger = NodeMerger(mock_db, mock_embedder)
    m1 = make_mention("Alan Turing", "PERSON")
    m2 = make_mention("John McCarthy", "PERSON")
    id1 = merger.resolve_entity(m1)
    id2 = merger.resolve_entity(m2)
    assert id1 != id2

# tests/unit/test_blind_spot_detector.py

def test_blind_spot_detects_high_sim_unconnected_nodes(mock_db, mock_vector_index):
    G = nx.Graph()
    G.add_node("node_privacy", surface_form="privacy", node_type="concept")
    G.add_node("node_surveillance", surface_form="surveillance", node_type="concept")
    # No edge between them

    mock_vector_index.query.return_value = [("node_surveillance", 0.88)]
    detector = BlindSpotDetector(mock_db, mock_vector_index, G)
    blind_spots = detector.detect()

    assert len(blind_spots) == 1
    assert blind_spots[0].semantic_similarity > 0.75
    assert blind_spots[0].graph_distance == -1  # No path

def test_blind_spot_skips_connected_nodes(mock_db, mock_vector_index):
    G = nx.Graph()
    G.add_node("node_a")
    G.add_node("node_b")
    G.add_edge("node_a", "node_b", weight=0.5)   # Already connected

    mock_vector_index.query.return_value = [("node_b", 0.90)]
    detector = BlindSpotDetector(mock_db, mock_vector_index, G)
    blind_spots = detector.detect()

    # Path length = 1, below threshold of 4 → not a blind spot
    assert len(blind_spots) == 0

18.3 Integration Tests

Python

# tests/integration/test_pipeline_end_to_end.py

def test_full_pipeline_markdown_to_graph(tmp_path, test_db, test_config):
    """
    Test: markdown file → ingest → extract → graph node created + searchable
    """
    md = tmp_path / "turing.md"
    md.write_text("# The Turing Test\n\nAlan Turing proposed the imitation game in 1950.")

    pipeline = ExtractionPipeline(test_config, test_db, graph_builder, vector_index)
    ingestor = MarkdownIngestor()
    docs = ingestor.ingest(str(md))
    pipeline.process(docs[0])

    # Node for "Alan Turing" should exist
    nodes = test_db.query("SELECT * FROM nodes WHERE surface_form LIKE '%Turing%'")
    assert len(nodes) > 0
    assert any("PERSON" in n["label"] for n in nodes)

    # Should be searchable via semantic search
    engine = SearchEngine(embedder, vector_index, test_db)
    results = engine.semantic_search("who proposed the imitation game")
    assert len(results) > 0
    assert any("Turing" in r["text"] for r in results)

def test_temporal_graph_at_returns_correct_state(test_db, sample_docs):
    """
    Test: graph_at(t1) does not contain nodes from documents created after t1
    """
    t1 = datetime(2022, 1, 1)
    t2 = datetime(2024, 1, 1)

    # Ingest two docs with different dates
    # ... (setup)

    temporal = TemporalGraph(test_db)
    G_at_t1 = temporal.graph_at(t1)
    G_at_t2 = temporal.graph_at(t2)

    assert len(G_at_t2.nodes) >= len(G_at_t1.nodes)

18.4 API Tests

Python

# tests/integration/test_api_graph.py

from fastapi.testclient import TestClient

def test_get_graph_returns_nodes_and_edges(client: TestClient, populated_db):
    resp = client.get("/api/v1/graph")
    assert resp.status_code == 200
    data = resp.json()
    assert "nodes" in data
    assert "edges" in data
    assert data["stats"]["node_count"] > 0

def test_semantic_search_returns_relevant_results(client: TestClient, populated_db):
    resp = client.post("/api/v1/search/semantic", json={"query": "neural networks", "k": 5})
    assert resp.status_code == 200
    results = resp.json()["results"]
    assert len(results) > 0
    assert all("score" in r for r in results)

def test_temporal_graph_at_endpoint(client: TestClient, populated_db):
    resp = client.get("/api/v1/temporal/graph_at?timestamp=2023-06-01T00:00:00Z")
    assert resp.status_code == 200
    data = resp.json()
    assert "nodes" in data
    assert "timestamp" in data

def test_blind_spots_endpoint(client: TestClient, populated_db):
    resp = client.get("/api/v1/insights/blind_spots")
    assert resp.status_code == 200
    blind_spots = resp.json()
    assert isinstance(blind_spots, list)
    if blind_spots:
        assert "semantic_similarity" in blind_spots[0]
        assert "graph_distance" in blind_spots[0]

19. Build, Packaging & Installation
19.1 Makefile

Makefile

.PHONY: all install dev build-ui test lint clean

# Install Python deps
install:
	pip install -e ".[dev]"
	python -m spacy download en_core_web_trf
	python -m spacy download en_core_web_sm

# Install UI deps
install-ui:
	cd ui && npm install

# Run dev mode (API + UI hot reload)
dev:
	make dev-api &
	make dev-ui

dev-api:
	uvicorn chronomap.api.server:app --host 127.0.0.1 --port 7331 --reload

dev-ui:
	cd ui && npm run dev

# Build production UI
build-ui:
	cd ui && npm run build
	cp -r ui/dist chronomap/api/static/

# Run all tests
test:
	pytest tests/unit/ -v --cov=chronomap --cov-report=term-missing

test-integration:
	pytest tests/integration/ -v --timeout=60

test-all:
	pytest tests/ -v

# Lint
lint:
	ruff check chronomap/
	mypy chronomap/
	cd ui && npm run check

# Format
format:
	ruff format chronomap/
	cd ui && npm run format

# First-time setup (generates config, downloads models, creates DB)
init:
	python -m chronomap.cli init

# Start ChronoMap
start:
	python -m chronomap.cli start

# Clean
clean:
	rm -rf dist/ ui/dist/ .pytest_cache/ __pycache__/ *.egg-info

# Package
build:
	python -m build
	cd ui && npm run build

19.2 pyproject.toml

toml

[project]
name = "chronomap"
version = "1.0.0"
description = "Local-first neural knowledge graph of your life"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.110",
    "uvicorn[standard]>=0.29",
    "spacy>=3.7",
    "sentence-transformers>=2.7",
    "keybert>=0.8",
    "bertopic>=0.16",
    "PyMuPDF>=1.24",
    "watchdog>=4.0",
    "networkx>=3.3",
    "numpy>=1.26",
    "scikit-learn>=1.4",
    "chromadb>=0.5",
    "pydantic>=2.7",
    "tomllib>=1.0",       # stdlib in 3.11+
    "python-multipart>=0.0.9",
    "icalendar>=5.0",
    "pyyaml>=6.0",
    "lumberjack>=0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "pytest-asyncio>=0.23",
    "httpx>=0.27",         # TestClient for FastAPI
    "ruff>=0.4",
    "mypy>=1.10",
]

[project.scripts]
chronomap = "chronomap.cli:main"

19.3 Installation Script

Bash

#!/bin/bash
# install.sh
set -e

CHRONOMAP_HOME="$HOME/.chronomap"
mkdir -p "$CHRONOMAP_HOME/logs" "$CHRONOMAP_HOME/vector_index"

echo "📥 Installing ChronoMap..."
pip install chronomap

echo "🧠 Downloading NLP models..."
python -m spacy download en_core_web_trf
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"

echo "⚙️ Generating initial config..."
chronomap init

echo "✅ ChronoMap installed."
echo "   Run 'chronomap start' to begin building your knowledge graph."
echo "   Dashboard: http://localhost:5173"
echo "   API:       http://localhost:7331"
echo ""
echo "💡 Add watch paths in: ~/.chronomap/config.toml → [daemon] watch_paths"

19.4 Docker Compose

YAML

# docker-compose.yml

version: '3.8'
services:
  chronomap:
    build: .
    ports:
      - "7331:7331"     # API
      - "5173:5173"     # UI
    volumes:
      - ~/.chronomap:/root/.chronomap
      - ~/Documents:/watch/Documents:ro     # Mount watched dirs (read-only)
      - ~/Downloads:/watch/Downloads:ro
    environment:
      - CHRONOMAP_CONFIG=/root/.chronomap/config.toml
    restart: unless-stopped

20. Platform Support Matrix
Feature	macOS Intel	macOS Apple Silicon	Linux amd64	Linux arm64	Windows amd64
Markdown/PDF Ingestion	✅	✅	✅	✅	✅
Browser Bookmark/History	✅	✅	✅	✅	✅
Email / Calendar / Ebook	✅	✅	✅	✅	✅
File Watch Daemon	✅ kqueue	✅ FSEvents	✅ inotify	✅ inotify	✅ ReadDirChanges
spaCy NER	✅	✅ MPS	✅	✅	✅
SentenceTransformers	✅	✅ MPS	✅ CUDA	✅	✅ CUDA
BERTopic	✅	✅	✅	✅	✅
sqlite-vec	✅	✅	✅	✅	✅
Chroma (alternative)	✅	✅	✅	✅	✅
FastAPI + Svelte UI	✅	✅	✅	✅	✅
GPU acceleration	✅ MPS	✅ MPS	✅ CUDA	⚠️	✅ CUDA
21. Performance Targets & Benchmarks
Metric	Target	Notes
Single file re-ingestion (< 1MB)	< 5s	Includes extraction + graph upsert
Chunk embedding (batch of 64)	< 2s CPU / < 0.3s GPU	SentenceTransformers batch
kNN search (100k vectors)	< 100ms	sqlite-vec or Chroma
Semantic search P99	< 500ms	Including hydration
Graph render (1000 nodes)	< 100ms	D3 force simulation
Graph render (50k nodes, cluster view)	< 200ms	LOD required, cluster aggregation
Temporal graph_at()	< 300ms	For graphs up to 100k nodes
Blind spot detection (10k nodes)	< 30s	Cached; kNN approach
Insight cache refresh	< 60s	Async, non-blocking
BERTopic fit (500 chunks)	< 20s CPU	Batch, async
SQLite upsert per node/edge	< 5ms	WAL mode
Daemon debounce latency	100ms	Configurable
21.1 Process Budget

text

Main processes:
├── chronomap-daemon     (Python, watchdog loop)
├── chronomap-api        (Python, uvicorn + FastAPI workers)
├── chronomap-ui         (Node/static files, Vite or pre-built)
└── [Shared]
    ├── NLP models in memory   (~500MB–2GB depending on spaCy model choice)
    ├── NetworkX graph         (~100MB for 50k nodes/200k edges)
    └── SQLite WAL             (on-disk, ~200MB for 100k chunks)

22. Error Handling Strategy
22.1 Error Categories

Python

# errors.py

class ChronoMapError(Exception):
    pass

class IngestionError(ChronoMapError):
    """File could not be read or parsed. Log and skip."""
    pass

class ExtractionError(ChronoMapError):
    """NLP model failed on a chunk. Log and skip chunk."""
    pass

class GraphError(ChronoMapError):
    """Graph upsert failed. Log, attempt retry once."""
    retryable = True

class VectorIndexError(ChronoMapError):
    """Vector index write failed. Log, queue for retry."""
    retryable = True

class InsightError(ChronoMapError):
    """Insight computation failed. Return cached result or empty list."""
    pass

class APIError(ChronoMapError):
    """API handler error. Return 500 with message."""
    pass

22.2 Core Policy: Errors Must Never Stop Ingestion

Python

# daemon/watch_daemon.py (error policy)

def _process(self, path: str):
    """
    Errors at any layer MUST be caught here.
    The daemon must never crash. Every file gets a best-effort attempt.
    Failure is logged with full traceback, then skipped.
    """
    try:
        ingestor = self.registry.find(path)
        if ingestor is None:
            return  # Silently skip unsupported types

        documents = ingestor.ingest(path)
        for doc in documents:
            try:
                self.pipeline.process(doc)
            except ExtractionError as e:
                log.warning(f"Extraction partial failure for {path}: {e}")
                # Partial results still written to DB where possible
            except GraphError as e:
                log.error(f"Graph update failed for {path}: {e}")
                # Retry once
                try:
                    self.pipeline.process(doc)
                except Exception:
                    log.error(f"Graph update retry failed for {path}", exc_info=True)

    except IngestionError as e:
        log.warning(f"Could not ingest {path}: {e}")
    except Exception as e:
        # Catch-all: never crash the daemon
        log.error(f"Unexpected error processing {path}: {e}", exc_info=True)

22.3 API Error Policy

Python

# api/middleware.py

from fastapi import Request
from fastapi.responses import JSONResponse

async def error_handler_middleware(request: Request, call_next):
    try:
        return await call_next(request)
    except InsightError as e:
        return JSONResponse(status_code=503, content={
            "error": "insight_unavailable",
            "message": "Insight computation failed. Try refreshing.",
            "detail": str(e)
        })
    except Exception as e:
        log.error(f"Unhandled API error: {e}", exc_info=True)
        return JSONResponse(status_code=500, content={
            "error": "internal_error",
            "message": "An unexpected error occurred."
        })

23. Dependency Registry
23.1 Python Dependencies

text

# requirements.txt

# API
fastapi>=0.110.0
uvicorn[standard]>=0.29.0
pydantic>=2.7.0
python-multipart>=0.0.9

# NLP
spacy>=3.7.0
sentence-transformers>=2.7.0
keybert>=0.8.0
bertopic>=0.16.0
scikit-learn>=1.4.0
numpy>=1.26.0
torch>=2.2.0               # SentenceTransformers backend

# Ingestion
PyMuPDF>=1.24.0            # PDF text + highlight extraction
watchdog>=4.0.0            # Cross-platform file watching
icalendar>=5.0.0           # .ics calendar parsing
pyyaml>=6.0.0              # YAML frontmatter in Markdown
mailparser>=3.15.0         # .eml parsing
beautifulsoup4>=4.12.0     # HTML bookmark file parsing

# Graph
networkx>=3.3.0
python-louvain>=0.16       # Community detection

# Storage
chromadb>=0.5.0            # Alternative vector index
sqlite-vec>=0.1.0          # Primary vector index

# Config
tomllib                    # stdlib >=3.11

# Logging
lumberjack>=0.0.0          # Log rotation

# Utilities
httpx>=0.27.0
uuid>=1.30

23.2 Frontend Dependencies

JSON

// ui/package.json

{
  "dependencies": {
    "svelte": "^4.x",
    "@sveltejs/kit": "^2.x",
    "d3": "^7.x",
    "three": "^0.x",
    "@threlte/core": "^7.x",
    "recharts": "^2.x",
    "date-fns": "^3.x",
    "lucide-svelte": "^0.x"
  },
  "devDependencies": {
    "typescript": "^5.x",
    "vite": "^5.x",
    "@sveltejs/vite-plugin-svelte": "^3.x",
    "tailwindcss": "^3.x",
    "svelte-check": "^3.x",
    "vitest": "^1.x",
    "@testing-library/svelte": "^4.x"
  }
}

23.3 External Requirements
Tool	Version	Required For	Install
Python	≥ 3.11	Backend	System package manager
Node.js	≥ 20	UI build	brew install node / nvm
pip	≥ 23	Python packages	python -m pip install --upgrade pip
spaCy model en_core_web_trf	Latest	High-accuracy NER	python -m spacy download en_core_web_trf
spaCy model en_core_web_sm	Latest	Low-RAM fallback	python -m spacy download en_core_web_sm
SentenceTransformers model	all-MiniLM-L6-v2	Embeddings	Auto-downloaded on first run
24. Milestone & Phased Rollout Plan
Phase 1 — Foundation (Weeks 1–4)

Goal: Working ingestor + basic searchable graph

    Project structure + pyproject.toml
    SQLite schema + migrations
    Document + TextChunk data models
    MarkdownIngestor + PDFIngestor
    IngestorRegistry
    Embedder (SentenceTransformers)
    ConceptExtractor (KeyBERT)
    VectorIndex (sqlite-vec backend)
    GraphBuilder (basic node upsert from concepts)
    GraphDB storage layer (nodes, chunks, mentions)
    FastAPI /api/v1/graph, /api/v1/search/semantic, /api/v1/health
    Minimal Svelte UI: search bar + result list
    Unit tests for ingestors + embedder + concept extractor

Deliverable: Ingest Markdown + PDFs. Ask a natural language question. Get back relevant chunks with document provenance.
Phase 2 — Full NLP + Knowledge Graph (Weeks 5–8)

Goal: Entities, relations, and a real graph visible in the UI

    EntityExtractor (spaCy NER)
    RelationExtractor (dependency parse)
    NodeMerger (string normalization + embedding dedup)
    EdgeScorer (co-mention + semantic weight)
    Full GraphBuilder (entity nodes, concept nodes, co-mention edges, relation edges)
    Node centrality computation (NetworkX)
    SQLite edges + mentions tables populated
    Node-level embeddings stored
    /api/v1/graph/node/:id + subgraph endpoint
    GraphCanvas.svelte (D3 force-directed render, colored by type)
    NodeDetail.svelte (click node → sources, relations)
    Unit tests for entity extractor, merger, scorer

Deliverable: A real, interactive knowledge graph of your notes + PDFs. Click any node to see all documents it came from.
Phase 3 — Watch Daemon + All Ingestors (Weeks 9–12)

Goal: Passive, continuous ingestion from all sources

    WatchDaemon (watchdog-based, debounce, hash check)
    BrowserIngestor (bookmarks + history)
    EmailIngestor (.mbox / .eml)
    CodeIngestor (Python, TypeScript, Go)
    EbookIngestor (Kindle clippings, Kobo)
    CalendarIngestor (.ics)
    Source deletion handler (last_seen update)
    Config file TOML loader
    Settings API + UI panel (watch paths, source toggles)
    Integration tests: file change → graph update

Deliverable: ChronoMap runs passively in the background. Drop a PDF, edit a note, export bookmarks — the graph updates automatically.
Phase 4 — Temporal Layer + Time Travel (Weeks 13–16)

Goal: Time-travel the graph via timeline slider

    TemporalGraph.graph_at(t)
    TemporalGraph.graph_diff(t1, t2)
    TemporalGraph.node_timeline(node_id)
    TemporalGraph.suggest_snapshots()
    /api/v1/temporal/* endpoints
    TimeSlider.svelte (UI timeline scrubber)
    BERTopic topic modeling + topic cluster coloring in UI
    ClusterView.svelte (zoom-out topic cluster view)
    Unit + integration tests for temporal queries

Deliverable: Scrub a timeline slider. Watch your knowledge graph evolve. See nodes appear and fade over time.
Phase 5 — Insight Engine (Weeks 17–20)

Goal: The "mirror of your mind" — blind spots, islands, obsessions, forgotten ideas

    BlindSpotDetector (kNN + path distance, scalable)
    IslandDetector (Louvain community detection)
    ObsessionTracker (windowed mention density + trend)
    ForgottenIdeas (dormant high-centrality nodes)
    InsightEngine with caching
    /api/v1/insights/* endpoints
    BlindSpotPanel.svelte
    IslandPanel.svelte
    ObsessionTracker.svelte (sparkline chart)
    ForgottenIdeas.svelte
    Insight cache invalidation on ingestion
    Unit tests for all insight detectors

Deliverable: ChronoMap tells you what you know but haven't connected, what you're obsessed with right now, and what great ideas you've forgotten.
Phase 6 — Polish & v1.0 (Weeks 21–24)

Goal: Production quality, installable, documented

    Three.js 3D graph mode (toggle)
    LOD (level-of-detail) for large graphs
    Installation script (macOS + Linux + Windows)
    Cross-platform testing
    Performance benchmarking vs targets
    80% test coverage
    Full user documentation
    chronomap init + chronomap start CLI
    Docker Compose setup
    Error handling audit (daemon never crashes)

Deliverable: v1.0 release. Install with one script. Run passively. Graph your life.
25. Open Questions & Future Work
25.1 Open Technical Questions
Question	Status	Notes
How to handle very large PDFs (500+ pages)?	Open	May need async chunking + page-level streaming
BERTopic refit strategy as corpus grows?	Open	Incremental UMAP is unstable; full refit periodically is safer but expensive
NetworkX memory ceiling for very large graphs?	Open	At 500k+ nodes may need graph DB (DGraph, ArangoDB) as backing store
sqlite-vec vs Chroma performance at 1M+ vectors?	Open	Benchmark needed; Chroma may win at scale
How to canonicalize entities across languages?	Backlog	Multi-lingual spaCy model required
Relation extraction quality vs noise tradeoff?	Open	Dep parse triples are noisy; co-mention edges may be safer default
How to handle duplicate documents (same content, different paths)?	Open	Content hash dedup at ingestion time; share nodes across doc_ids
25.2 Potential Future Modules
Module	Description	Priority
Graph export	Export to Obsidian-compatible markdown + GEXF/GraphML for Gephi	High
Spaced repetition integration	"Forgotten idea" → Anki card	High
Multi-language support	spaCy multilingual models	Medium
Audio/video ingestion	Whisper transcription → ingest	Medium
Web clipper browser extension	One-click "add to ChronoMap"	High
LLM-powered insight narration	Local LLM describes blind spots in prose	Medium
Collaborative graph merge	Merge two local graphs (family/team use)	Low
Graph-powered writing assistant	"What do I know about X?" → outline	High
Mobile app (read-only)	Browse your graph on iOS/Android	Low
Custom concept taxonomy	User-defined concept categories + ontology	Medium
25.3 Known Limitations at v1.0

    Entity resolution is imperfect. The merger uses embedding similarity with a threshold — it will sometimes merge distinct entities (false positive) or split the same entity (false negative). Threshold tuning is corpus-dependent.

    Relation extraction is noisy. Dependency parse SVO triples produce many low-confidence edges. The graph is most reliable for co-mention edges; relation-typed edges should be treated as supplementary.

    BERTopic requires a minimum corpus. The min_topic_size constraint means topic modeling doesn't activate until you have at least ~100–200 chunks ingested. Small corpora will have most chunks assigned to the outlier topic (-1).

    PDF quality varies dramatically. Scanned PDFs (image-only), badly structured PDFs, or those with custom encoding will yield poor text extraction. PyMuPDF handles most standard PDFs well, but the user should be aware of this limitation.

    Graph rendering at scale requires LOD. Rendering 50,000 nodes in a browser force simulation without clustering will cause frame drops. The ClusterView mode (topic-level aggregation) is required for large graphs.

    The temporal model assumes document created_at as truth. If a file's filesystem creation time is wrong (e.g., copied from backup), the temporal graph may be inaccurate. The user can override timestamps via settings.

    No graph editing. ChronoMap is read-only and fully automated. Users who want to add manual connections or override NLP decisions cannot do so in v1.0.

End of ChronoMap Comprehensive Engineering Design Document — v1.0

This document is fully self-contained. All AI processing is local. No data leaves the machine. The graph is yours.
