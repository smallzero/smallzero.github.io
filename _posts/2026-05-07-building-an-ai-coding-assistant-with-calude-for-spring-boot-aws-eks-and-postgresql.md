---
layout: post
title: Building an AI Coding Assistant with Claude for Spring Boot, AWS EKS & PostgreSQL
categories: Programming
excerpt: A step-by-step guide to creating a domain-expert AI assistant powered by Claude (Anthropic), with optional OpenClaw agent framework. Focus building an AI Coding Assistant for Spring Boot, AWS EKS and PostgreSQL
---
# Building an AI Coding Assistant with Claude for Spring Boot, AWS EKS & PostgreSQL

> A step-by-step guide to creating a domain-expert AI assistant powered by Claude (Anthropic), with optional OpenClaw agent framework.

---

## Architecture Overview

```
Developer (You)
    ↓ (chat / IDE / Slack)
OpenClaw Agent Runtime  ←→  Claude API (Anthropic)
    ↓                            ↑
Vector DB (RAG)          Knowledge Base
(Pinecone/Weaviate)      (Spring Boot, EKS, PostgreSQL docs)
```

---

## Step 1: Get Your Claude API Access

1. Sign up at [Anthropic Console](https://console.anthropic.com/)
2. Generate an **API key**
3. Choose a model (e.g., `claude-3-opus` or `claude-sonnet`)

---

## Step 2: Build a Domain Knowledge Base

Curate documents so Claude has **strong, specific knowledge**:

| Domain | Sources to Collect |
|---|---|
| **Spring Boot** | Official docs, Spring Security, Spring Data JPA, common patterns, your team's code templates |
| **AWS EKS** | EKS best practices guide, Helm charts, IAM/IRSA setup, cluster autoscaler, ALB ingress |
| **PostgreSQL DBA** | pg_stat, VACUUM, indexing strategies, connection pooling (HikariCP/PgBouncer), replication, backup/restore |

**Format:** Split into small markdown chunks (~500–1000 tokens each) with metadata tags.

---

## Step 3: Set Up RAG (Retrieval-Augmented Generation)

This is what gives Claude **deep, accurate** domain knowledge beyond its training.

### Components

1. **Choose a Vector Database:** Pinecone, Weaviate, pgvector (you can use PostgreSQL itself!), or OpenSearch
2. **Embed your documents** using an embedding model (e.g., Cohere, OpenAI `text-embedding-3`, or Voyage AI)
3. **At query time:**
   - Embed the user's question
   - Retrieve top-k relevant chunks from the vector DB
   - Inject them into Claude's prompt as context

### Sample RAG Flow (Python)

```python
import anthropic

client = anthropic.Anthropic(api_key="YOUR_KEY")

def ask_assistant(question):
    # 1. Retrieve relevant docs from vector DB
    context_chunks = vector_db.search(embed(question), top_k=5)
    context = "\n---\n".join(context_chunks)

    # 2. Call Claude with context
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        system="""You are an expert coding assistant specializing in:
- Spring Boot (including Security, Data JPA, Actuator)
- AWS EKS (deployment, scaling, networking, IAM)
- PostgreSQL DBA (performance tuning, indexing, replication, backup)
Use the provided context to give accurate, actionable answers.""",
        messages=[{
            "role": "user",
            "content": f"Context:\n{context}\n\nQuestion: {question}"
        }]
    )
    return response.content[0].text
```

---

## Step 4 (Optional): Use OpenClaw for a Full Agent Experience

[OpenClaw](https://openclaw.im/) turns Claude into an autonomous, persistent AI agent that can:

- Execute shell commands, manage files, browse the web
- Maintain memory across sessions
- Work via Slack, Discord, WhatsApp, or terminal
- Run scheduled tasks (like a cron job)

### Quick Setup

```bash
# Install OpenClaw
npm install -g openclaw
# or
curl -sSL https://openclaw.im/install.sh | bash

# Initialize with Claude
openclaw init --model claude --api-key YOUR_ANTHROPIC_KEY

# Add custom skills for your stack
openclaw skill add spring-boot-helper
openclaw skill add eks-deployer
openclaw skill add postgres-dba
```

You can write **custom skills** (a folder with code + `SKILL.md`) so your agent knows how to:

- Generate Spring Boot boilerplate
- Run `kubectl` commands against your EKS cluster
- Execute `psql` queries for DB administration

---

## Step 5: Technology Stack Summary

| Layer | Technology |
|---|---|
| **LLM** | Claude (Anthropic API) |
| **Agent Framework** | OpenClaw (optional) or custom Python/Node.js app |
| **Knowledge Store** | Vector DB — pgvector, Pinecone, or Weaviate |
| **Embedding** | Voyage AI, Cohere, or OpenAI embeddings |
| **Backend** | Spring Boot (your app) or Python FastAPI (for the assistant service) |
| **Deployment** | AWS EKS (same cluster as your apps) |
| **Database** | PostgreSQL (for app data + optionally pgvector for RAG) |
| **Interface** | IDE plugin, Slack bot, CLI, or web UI |

---

## Step 6: Deploy on AWS EKS

1. **Containerize** your assistant service (Dockerfile)
2. **Deploy** to your EKS cluster via Helm chart or Kubernetes manifests
3. Store the Anthropic API key in **AWS Secrets Manager** or K8s Secrets
4. Expose via **ALB Ingress** or internal service for your team

---

## Step 7: Iterate & Improve

- **Log unanswered questions** → add missing docs to knowledge base
- **Fine-tune the system prompt** for your team's coding standards
- **Add more skills** (CI/CD helpers, code review automation, DB migration scripts)

---

## Quick Start Recommendation

| Path | When to Use |
|---|---|
| **Simple RAG App** | Get Claude API key → Python RAG service → load docs into pgvector → deploy on EKS |
| **Full Agent (OpenClaw)** | Need file access, shell commands, persistent memory, and chat platform integration |

---

## References

- [Anthropic API Docs](https://docs.anthropic.com/claude/docs/)
- [RAG Patterns (Pinecone)](https://www.pinecone.io/learn/retrieval-augmented-generation/)
- [Spring Boot Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [OpenClaw Official Site](https://openclaw.im/)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)

---

*Published: 2026-05-07*