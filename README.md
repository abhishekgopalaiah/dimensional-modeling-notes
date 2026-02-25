# Data Warehouse Interview Preparation Guide

> **Target**: Senior Data Engineer / Data Architect roles (8+ years experience)  
> **Status**: Interview-ready materials  
> **Last Updated**: February 2026

---

## 📚 How to Use This Guide

### For Interview Preparation (Recommended Order)

1. **Start Here**: `09_Quick_Reference.md` (15 min) - Get the big picture
2. **Core Foundation**: `01_Core_Concepts.md` (45 min) - Master fundamentals
3. **Dimensions Deep Dive**: `02_dimensions.md` (60 min) - All dimension types
4. **Facts Deep Dive**: `03_facts.md` (90 min) - All fact table patterns
5. **Bridge Tables**: `04_Bridge_Tables.md` (30 min) - Many-to-many relationships
6. **Temporal Logic**: `06_SCD impact on facts.md` (45 min) - SCD and late data
7. **ETL Patterns**: `08_incremental loading strategies.md` + `09_partitioning and performance.md` (60 min)
8. **Practice**: `08_Interview_QA.md` (120 min) - 50 Q&A with model answers

**Total Time**: ~7-8 hours for comprehensive preparation

### For Last-Minute Review (1 Hour Before Interview)

1. `09_Quick_Reference.md` - Cheat sheet (15 min)
2. `08_Interview_QA.md` - Skim top 20 questions (30 min)
3. Review your own project examples (15 min)

---

## 📁 File Structure & Purpose

### ✅ Core Interview Files (Start Here)

| File | Purpose | Time | Priority |
|------|---------|------|----------|
| **09_Quick_Reference.md** | Cheat sheet for last-minute review | 15 min | 🔥 CRITICAL |
| **08_Interview_QA.md** | 50 interview questions with model answers | 2 hours | 🔥 CRITICAL |
| **01_Core_Concepts.md** | Entities, facts, grain, additivity | 45 min | 🔥 CRITICAL |

### 📖 Deep Dive Files (Topic Mastery)

| File | Topic | Time | When to Read |
|------|-------|------|--------------|
| **02_dimensions.md** | All dimension types, SCD, conformed dims | 60 min | After core concepts |
| **03_facts.md** | Transaction, snapshot, accumulating facts | 90 min | After dimensions |
| **04_Bridge_Tables.md** | Many-to-many relationships, allocation | 30 min | After facts |
| **06_SCD impact on facts.md** | Temporal correctness, point-in-time joins | 45 min | After dimensions |
| **07_late_arrving_data.md** | Late facts, retroactive corrections | 30 min | After SCD |

### 🔧 ETL & Engineering Files

| File | Topic | Time | When to Read |
|------|-------|------|--------------|
| **08_incremental loading strategies.md** | Append, MERGE, CDC, partition overwrite | 30 min | Before interviews |
| **09_partitioning and performance.md** | Partitioning, clustering, optimization | 30 min | Before interviews |
| **10_multi-fact constellation.md** | Enterprise architecture patterns | 30 min | For architect roles |

### 📊 Specialized Topics

| File | Topic | Time | When to Read |
|------|-------|------|--------------|
| **01_Grains.md** | Grain discipline deep dive | 30 min | If grain questions arise |
| **04_Factless facts.md** | Coverage facts, event tracking | 20 min | If asked about factless |
| **05_feature engineering.md** | ML context for dimensional models | 30 min | For ML-adjacent roles |
| **periodic_snapshot.md** | Snapshot facts detailed guide | 45 min | If snapshot questions arise |

### 🗑️ Archive (Reference Only)

| File | Status | Action |
|------|--------|--------|
| **12_wht_nest.md** | Interview strategy notes | Keep for reference |

---

## 🎯 Interview Preparation Paths

### Path 1: Quick Prep (4 Hours)

For interviews in 1-2 days:

1. ✅ `09_Quick_Reference.md` (15 min)
2. ✅ `01_Core_Concepts.md` (45 min)
3. ✅ `08_Interview_QA.md` - Q1-Q30 (90 min)
4. ✅ `02_dimensions.md` - SCD sections only (30 min)
5. ✅ `03_facts.md` - Fact types section only (30 min)
6. ✅ Review your own projects (30 min)

### Path 2: Comprehensive Prep (8 Hours)

For interviews in 1 week:

**Day 1-2**: Foundation
- `01_Core_Concepts.md`
- `02_dimensions.md`
- `03_facts.md`

**Day 3-4**: Advanced Topics
- `04_Bridge_Tables.md`
- `06_SCD impact on facts.md`
- `07_late_arrving_data.md`

**Day 5-6**: ETL & Practice
- `08_incremental loading strategies.md`
- `09_partitioning and performance.md`
- `08_Interview_QA.md` (all 50 questions)

**Day 7**: Review & Mock
- `09_Quick_Reference.md`
- Mock interview with friend
- Prepare project examples

### Path 3: Architect-Level Prep (12 Hours)

For lead/architect roles:

Complete Path 2, then add:
- `10_multi-fact constellation.md`
- `05_feature engineering.md`
- `periodic_snapshot.md`
- Focus on trade-off discussions
- Prepare architecture diagrams

---

## 🎓 What You'll Master

### Core Competencies

- ✅ **Grain Discipline**: Define and validate fact table grain
- ✅ **Dimension Design**: SCD types, conformed dimensions, bridge tables
- ✅ **Fact Modeling**: Transaction, snapshot, accumulating patterns
- ✅ **Temporal Logic**: Point-in-time joins, late-arriving data
- ✅ **ETL Patterns**: Incremental loading, idempotency, CDC
- ✅ **Performance**: Partitioning, clustering, optimization

### Interview Skills

- ✅ Answer 50 common questions confidently
- ✅ Design dimensional models from requirements
- ✅ Debug data quality issues
- ✅ Discuss trade-offs intelligently
- ✅ Explain complex concepts simply
- ✅ Reference real-world experience

---

## 📊 Coverage Matrix

### Topics vs Files

| Topic | Core Concepts | Dimensions | Facts | Bridge | SCD | ETL | Interview QA |
|-------|--------------|------------|-------|--------|-----|-----|--------------|
| **Grain** | ✅✅✅ | ✅ | ✅ | | | | ✅ |
| **SCD Type 2** | | ✅✅✅ | ✅ | | ✅✅✅ | | ✅✅ |
| **Fact Types** | ✅✅ | | ✅✅✅ | | | | ✅✅ |
| **Additivity** | ✅✅✅ | | ✅✅ | | | | ✅✅ |
| **Bridge Tables** | | ✅ | ✅ | ✅✅✅ | | | ✅✅ |
| **Incremental Load** | | | | | | ✅✅✅ | ✅✅ |
| **Partitioning** | | | ✅ | | | ✅✅✅ | ✅✅ |
| **Late Data** | | | ✅ | | ✅✅✅ | ✅✅ | ✅✅ |

Legend: ✅ = Covered, ✅✅ = Detailed, ✅✅✅ = Comprehensive

---

## 🔥 Must-Know for 80% of Interviews

Based on `12_wht_nest.md` analysis, focus on these 10 topics:

1. ⭐ **Star schema vs snowflake** → `01_Core_Concepts.md`, `02_dimensions.md`
2. ⭐ **Surrogate vs natural keys** → `01_Core_Concepts.md`, `02_dimensions.md`
3. ⭐ **SCD Types (especially Type 2)** → `02_dimensions.md`, `06_SCD impact on facts.md`
4. ⭐ **Handling late arriving dimensions** → `07_late_arrving_data.md`
5. ⭐ **Junk vs degenerate dimensions** → `02_dimensions.md`
6. ⭐ **Conformed dimensions** → `02_dimensions.md`, `10_multi-fact constellation.md`
7. ⭐ **Bridge tables** → `04_Bridge_Tables.md`
8. ⭐ **Mini-dimension concept** → `02_dimensions.md`
9. ⭐ **How fact joins with SCD2 dimension** → `06_SCD impact on facts.md`
10. ⭐ **Handling null/unknown dimension rows** → `02_dimensions.md`

**Master these 10 topics and you're above average.**

---

## 💡 Study Tips

### Active Learning

- ✅ Don't just read—practice explaining out loud
- ✅ Draw diagrams for each pattern
- ✅ Write SQL examples from memory
- ✅ Relate concepts to your own projects
- ✅ Teach concepts to a colleague/friend

### Spaced Repetition

- Day 1: Read core concepts
- Day 3: Review core concepts + new topics
- Day 5: Review all previous + new topics
- Day 7: Full review + mock interview

### Mock Interviews

- Practice with `08_Interview_QA.md` questions
- Time yourself (2-3 min per answer)
- Record yourself and review
- Focus on clarity and structure
- Prepare follow-up examples

---

## 🚨 Common Pitfalls to Avoid

### During Study

- ❌ Trying to memorize everything
- ❌ Skipping hands-on practice
- ❌ Not relating to real projects
- ❌ Ignoring the "why" behind concepts
- ❌ Not testing your understanding

### During Interview

- ❌ Jumping to solutions without asking questions
- ❌ Using jargon without explaining
- ❌ Not discussing trade-offs
- ❌ Claiming to know everything
- ❌ Giving textbook answers without context

---

## 📈 Progress Tracking

### Checklist: Core Concepts

- [ ] Can explain grain in one sentence
- [ ] Can describe all three fact types
- [ ] Understand additivity rules
- [ ] Know when to use surrogate keys
- [ ] Can validate fact table design

### Checklist: Dimensions

- [ ] Can implement SCD Type 2
- [ ] Understand point-in-time joins
- [ ] Know all major dimension types
- [ ] Can design conformed dimensions
- [ ] Understand mini-dimension pattern

### Checklist: Advanced

- [ ] Can design bridge tables
- [ ] Understand incremental loading
- [ ] Know partitioning strategies
- [ ] Can handle late-arriving data
- [ ] Understand multi-fact constellation

### Checklist: Interview Ready

- [ ] Can answer top 20 questions confidently
- [ ] Have 3-5 project examples ready
- [ ] Can draw dimensional models on whiteboard
- [ ] Comfortable discussing trade-offs
- [ ] Can explain concepts to non-technical people

---

## 🎯 Final Preparation Checklist

### 1 Week Before

- [ ] Complete Path 2 (Comprehensive Prep)
- [ ] Practice all 50 interview questions
- [ ] Prepare project examples
- [ ] Review company's tech stack
- [ ] Research interviewer backgrounds

### 1 Day Before

- [ ] Review `09_Quick_Reference.md`
- [ ] Skim `08_Interview_QA.md` top 30 questions
- [ ] Review your project examples
- [ ] Get good sleep
- [ ] Prepare questions for interviewer

### 1 Hour Before

- [ ] Review `09_Quick_Reference.md` (15 min)
- [ ] Review your notes on company (10 min)
- [ ] Mental preparation (10 min)
- [ ] Arrive early / test video setup

---

## 🤝 Contributing & Feedback

### Keep Updated

- Add new questions you encounter in interviews
- Update with new patterns from real projects
- Refine answers based on feedback
- Add company-specific notes

### Version History

- **v1.0** (Feb 2026): Initial consolidated version
  - 9 core files created
  - 50 interview questions added
  - Quick reference guide added

---

## 📞 Quick Links

### Most Important Files

1. [Quick Reference](09_Quick_Reference.md) - Start here
2. [Interview Q&A](08_Interview_QA.md) - Practice questions
3. [Core Concepts](01_Core_Concepts.md) - Foundation

### By Topic

- **Dimensions**: [02_dimensions.md](02_dimensions.md)
- **Facts**: [03_facts.md](03_facts.md)
- **Bridge Tables**: [04_Bridge_Tables.md](04_Bridge_Tables.md)
- **SCD & Temporal**: [06_SCD impact on facts.md](06_SCD%20impact%20on%20facts.md)
- **ETL**: [08_incremental loading strategies.md](08_incremental%20loading%20strategies.md)

---

## 🎓 Success Metrics

You're ready for interviews when you can:

1. ✅ Explain grain discipline without hesitation
2. ✅ Design a dimensional model in 15 minutes
3. ✅ Implement SCD Type 2 on a whiteboard
4. ✅ Debug a grain violation in sample data
5. ✅ Discuss trade-offs between star and snowflake
6. ✅ Answer "why" questions, not just "what"
7. ✅ Reference real projects naturally
8. ✅ Ask clarifying questions before designing
9. ✅ Admit knowledge gaps confidently
10. ✅ Explain complex concepts simply

---

**Good luck with your interviews! Remember: They're evaluating how you think, not just what you know.**

*For questions or updates, maintain this README as your central navigation hub.*
