<a id="top"></a>

# 📚 Tech Knowledge Base

A curated collection of full courses, notes, cheatsheets, and interview-prep Q&A covering AI/ML, Databases, DevOps, Backend, DSA, System Design, and more.

This repository serves as a personal reference hub and quick revision guide for interviews, projects, and day-to-day development.

`27 structured courses` · `12 topic areas` · continuously updated

---

## 📑 Contents

[AI-ML](#ai-ml) · [Databases](#databases) · [DevOps](#devops) · [Interview Q&A](#interview-qa) · [Backend](#backend) · [Cloud](#cloud) · [DSA](#dsa) · [Git & GitHub](#git--github) · [Python](#python) · [MindMaps](#mindmaps) · [Security](#security) · [System Design](#system-design) · [Prompts](#prompts) · [Others](#others)

---

## 🚀 At a Glance

| Area | Highlights |
|---|---|
| 🤖 [AI-ML](#ai-ml) | 7 courses — LLM fundamentals through vLLM production serving |
| 🗄️ [Databases](#databases) | 6 courses — PostgreSQL, MongoDB, Redis, ClickHouse, MinIO, Alembic |
| ⚙️ [DevOps](#devops) | 11-course learning path (Linux → Security & DevSecOps) + standalone Docker course |
| 🎯 [Interview Q&A](#interview-qa) | 11 files, Generative AI through LLMOps |
| 🧩 [Backend](#backend) | OAuth 2.0 course + API/JWT references |
| 🐍 [Python](#python) | uv course + logging, testing, and async notes |
| 🧠 [DSA](#dsa) | Pattern guides, revision sheets, and cheatsheets |
| 🗺️ [MindMaps](#mindmaps) | Visual maps for AI engineering, LLD, and DSA patterns |

---

## Sections Overview

<a id="ai-ml"></a>

### 🤖 AI-ML

**Courses**
| Course | About |
|---|---|
| [LLM Fundamentals](./AI-ML/llm-fundamentals-course/readme.md) | Core concepts behind large language models |
| [LangChain Core](./AI-ML/langchain-core-course/00-index.md) | LLM/FastAPI engineer → production LCEL practitioner |
| [LangGraph](./AI-ML/langgraph-course/00-index.md) | Orchestration novice → production expert |
| [DeepAgents](./AI-ML/deepagents-course/00-index.md) | LangGraph practitioner → production agent-runtime architect |
| [RAG](./AI-ML/rag-course/00-index.md) | Vector basics → agentic, enterprise-grade RAG systems |
| [MCP](./AI-ML/mcp-course/00-index.md) | Protocol fundamentals → production agent infrastructure |
| [vLLM](./AI-ML/vllm-course/00-index.md) | Inference engineering, fundamentals → production serving |

**Notes & References**
- [Writing Skills Guide](./AI-ML/writing-skills-guide.md)
- [Hugging Face Cache Explained](./AI-ML/huggingface_cache_explained.md)
- [LLM Generation Parameters](./AI-ML/llm_generation_parameters.md)
- [LLM Cheatsheet (PDF)](https://drive.google.com/file/d/1ak_KaAs3HbvvXVKqt4OLvBfRo9UXFXff/view?usp=sharing)
- [AI Book (PDF)](https://drive.google.com/file/d/14KP15RxNxZmiZSfV1biQOTdZQOb5J6nT/view?usp=sharing)

[⬆ Back to top](#top)

<a id="databases"></a>

### 🗄️ Databases
- [PostgreSQL](./Databases/postgresql-course/00-index.md)
- [MongoDB & the Aggregation Pipeline](./Databases/mongodb-course/readme.md)
- [Redis](./Databases/redis-course/00-index.md)
- [ClickHouse & Columnar Databases](./Databases/clickhouse-course/00-index.md)
- [MinIO & Object Storage](./Databases/minio-course/00-index.md)
- [Alembic & Database Schema Migrations](./Databases/alembic-course/00-index.md)

[⬆ Back to top](#top)

<a id="devops"></a>

### ⚙️ DevOps

- [DevOps Engineering — Complete Learning Path](./DevOps_Book/README.md) — 11 courses, zero to production-grade DevOps engineer
  - [01 · Linux Fundamentals](./DevOps_Book/01-Linux-Fundamentals/00-index.md) — 17 chapters
  - [02 · Networking Basics](./DevOps_Book/02-Networking-Basics/00-index.md) — 16 chapters
  - [03 · Git & Version Control](./DevOps_Book/03-Git-Version-Control/00-index.md) — 17 chapters
  - [04 · Docker](./DevOps_Book/04-Docker/00-index.md) — 17 chapters
  - [05 · CI/CD Pipelines](./DevOps_Book/05-CI-CD-Pipelines/00-index.md) — 17 chapters
  - [06 · Cloud Fundamentals (AWS)](./DevOps_Book/06-Cloud-Fundamentals-AWS/00-index.md) — 17 chapters
  - [07 · Infrastructure as Code (Terraform)](./DevOps_Book/07-Infrastructure-as-Code-Terraform/00-index.md) — 17 chapters
  - [08 · Kubernetes Basics](./DevOps_Book/08-Kubernetes-Basics/00-index.md) — 19 chapters
  - [09 · Advanced Kubernetes](./DevOps_Book/09-Advanced-Kubernetes/00-index.md) — 18 chapters
  - [10 · Monitoring & Logging](./DevOps_Book/10-Monitoring-and-Logging/00-index.md) — 18 chapters
  - [11 · Security & DevSecOps](./DevOps_Book/11-Security-DevSecOps/00-index.md) — 18 chapters
- [Docker Course (standalone)](./DevOps/docker-course/00-roadmap.md) — 10 chapters, beginner to expert

**Other DevOps references**
- [Roadmap_Devops.md](./DevOps/Roadmap_Devops.md)
- [helm_kubernetes_guide.md](./DevOps/helm_kubernetes_guide.md)
- [kubernetes_commands.md](./DevOps/kubernetes_commands.md)
- [kubernetes_interview_qa.md](./DevOps/kubernetes_interview_qa.md)
- [Essential_Network_Ports.jpg](./DevOps/Essential_Network_Ports.jpg)
- [HTTP_Status_Code.jpg](./DevOps/HTTP_Status_Code.jpg)
- [Docker_Book.pdf](https://drive.google.com/file/d/1jb-N1CR3bxw5IQPmGw4zBnSJvHFW1M7J/view?usp=sharing)
- [Docker_Tricks.pdf](https://drive.google.com/file/d/1j7OShQR3Zm4GkhN2KTGEiAR87cjXReON/view?usp=sharing)

[⬆ Back to top](#top)

<a id="interview-qa"></a>

### 🎯 Interview Q&A
- [AI/ML Interview Q&A](./Interview-QA/readme.md) — 11 files covering Generative AI through LLMOps

<a id="backend"></a>

### 🧩 Backend
- [OAuth 2.0 & Token-Based Authentication Course](./Backend/oauth2-course/00-index.md)
- [JWT_Token.md](./Backend/JWT_Token.md)
- [20API_Concepts.jpg](./Backend/20API_Concepts.jpg)
- [API_Architecture_Types.jpg](./Backend/API_Architecture_Types.jpg)
- [RestAPI.jpg](./Backend/RestAPI.jpg)
- [API_Basics.pdf](https://drive.google.com/file/d/1iOgVqmyFurQ9hM5w7ZviYs5Q3cm3DpmJ/view?usp=sharing)

<a id="cloud"></a>

### ☁️ Cloud
- [AWS_Roadmap.jpg](./Cloud/AWS_Roadmap.jpg)

<a id="dsa"></a>

### 🧠 DSA
- [Sorting_Algo.jpg](./DSA/Sorting_Algo.jpg)
- [DSA_Pattern_Guide.pdf](https://drive.google.com/file/d/1zbCxFN7EFADrk1fA60qk7cN323oRHjdt/view?usp=sharing)
- [DSA_Quick_Revision_Guide.pdf](https://drive.google.com/file/d/1_437gzKZPDcX1gp7kaYWYAZ4rcQK0f82/view?usp=sharing)
- [Logic_Building_&_Guess_The_Output.pdf](https://drive.google.com/file/d/1YPtb6ICcoAIGNkvCqtTmyUBI_cWFaNQ5/view?usp=sharing)
- [Trees_Notes_-1.pdf](https://drive.google.com/file/d/12Hm0MSBHa-fpvU2B1o_8W_gSz7-kdP-f/view?usp=sharing)
- [100_DSA_Questions_All_Patterns.docx](https://docs.google.com/document/d/1KO8OwklhTLUPvUS0gBccITxvs44ScKX7/mobilebasic)
- [Coding Patterns for Solving DSA Problems](https://github.com/Chanda-Abdul/Several-Coding-Patterns-for-Solving-Data-Structures-and-Algorithms-Problems-during-Interviews)

[⬆ Back to top](#top)

<a id="git--github"></a>

### 🔧 Git & GitHub
- [Github-cli-guide.md](./Git-GitHub/Github-cli-guide.md)
- [Git_Basics.md](./Git-GitHub/Git_Basics.md)
- [Git_Helper_Files.md](./Git-GitHub/Git_Helper_Files.md)
- [Git_LICENSE_Guide.md](./Git-GitHub/Git_LICENSE_Guide.md)

<a id="python"></a>

### 🐍 Python
- [uv — The Complete Python Package & Project Manager Course](./Python/uv-course/00-index.md)
- [EmailMessage-vs-MIMEMultipart.md](./Python/EmailMessage-vs-MIMEMultipart.md)
- [logging_basics.md](./Python/logging_basics.md)
- [motor-vs-pymongo.md](./Python/motor-vs-pymongo.md)
- [pytest_basic.md](./Python/pytest_basic.md)

<a id="mindmaps"></a>

### 🗺️ MindMaps
- [AI Engineer Complete MindMap](https://whimsical.com/akki-s-workspace2/ai-engineer-mindmap-GnDcMXsXrt8biX2xM3pZAS)
- [Low Level Design (LLD)](https://whimsical.com/akki-s-workspace2/lld-2ybEBbrajSK4BiPBNFdfQP)
- [DSA Patterns](https://whimsical.com/akki-s-workspace2/dsa-patterns-JEvwrbR2rLP88DLyhqX5zj)

<a id="security"></a>

### 🔐 Security
- Security-related concepts (to be expanded)

<a id="system-design"></a>

### 🏗️ System Design
- System design concepts and resources

<a id="prompts"></a>

### 📝 Prompts
- Prompt engineering references and examples

<a id="others"></a>

### 📦 Others
- [Final_year_project.pdf](https://drive.google.com/file/d/1Rxc9cN7V8wWy7JUoYx8tP4ZTwL7tZHU7/view?usp=sharing)

[⬆ Back to top](#top)

---

## 🎯 Purpose

- 📖 Quick revision before interviews
- 🧠 Consolidated learning in one place
- 🔍 Easy reference for real-world projects
- ⚡ Save time searching scattered resources

---

## 🛠️ How to Use

- Jump to a section via the [Contents](#-contents) links, or browse folders directly
- Course folders use numbered chapters (`00-index.md`/`readme.md`, `01-...`, `02-...`) — start at the index and read in order
- Open `.md`, `.pdf`, or `.jpg` files as needed
- Use it as a second brain for tech concepts

---

## 🌱 Future Improvements

- Expand System Design & Security sections
- Add real-world case studies
- Improve categorization and tagging

---

## ⭐ Contributing

This is a personal knowledge base, but feel free to:
- Suggest improvements
- Share better resources
- Raise issues

---

## 📌 Note

This repo is continuously evolving as I learn and explore new technologies.
