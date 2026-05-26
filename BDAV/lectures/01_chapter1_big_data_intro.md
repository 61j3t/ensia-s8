# Chapter 1 — Understanding Big Data

## Bird's eye view

- **Big Data** = data whose size, speed, or type breaks traditional RDBMS — needs new tools to capture, store, process.
- Characterized by the **7 Vs**: Volume, Velocity, Variety (the original 3 by Laney 2001) → Veracity, Value, Variability, Visualization (added later).
- A dataset doesn't need *all* 7 Vs to be "big data" — even one significant V is enough.
- **4 types of analytics** ladder up: Descriptive (what happened) → Diagnostic (why) → Predictive (what might) → Prescriptive (what to do).
- Big Data powers: social media analytics, IoT, e-commerce recommendations, healthcare, finance, agriculture, smart cities.
- Career paths: Big Data Architect, Data Engineer, Data Scientist, AI Engineer, Big Data Consultant.
- The chapter sets up the *problem* (volume/velocity/variety beyond RDBMS); subsequent chapters give the *solution* (Hadoop ecosystem).

---

## 1. What is Big Data?

Three reference definitions converge on the same idea:

- **IBM**: data sets whose size or type is beyond the ability of traditional relational databases to capture, manage, and process with low latency. Includes **structured, semi-structured, and unstructured** data.
- **Gartner**: high-volume, high-velocity, high-variety information assets that demand cost-effective, innovative processing for enhanced insight and decision-making.
- **Techopedia**: large and complex data sets, especially from new sources, that traditional data processing software cannot manage.

Common thread: **Volume + Velocity + Variety + need for new tools**.

Scale figures (used to motivate the field):
- ~3.5 quintillion bytes generated daily (2023)
- Big Data market projected $103 B by 2027 (CAGR 10.48%)
- Analytics industry expected to hit $650 B by 2029
- Data science jobs +28% by 2026 → ~11.5 M new positions
- Hadoop market: ~$842 B by 2030

---

## 2. The 7 Vs (the chapter's centerpiece)

| V | Year/Author | Definition | Main challenges |
|---|---|---|---|
| **Volume** | Laney, 2001 | Massive size of datasets generated every second globally | Traditional storage can't scale; redundancy and availability get complex |
| **Velocity** | Laney, 2001 | Speed at which data is generated, collected, processed (real-time or near-real-time) | Processing streams in real-time; low-latency response (e.g., fraud detection) |
| **Variety** | Laney, 2001 | Diversity of formats and sources — structured, semi-structured, unstructured | Integrating disparate types; processing unstructured data effectively |
| **Veracity** | IBM, 2012 | Quality, accuracy, and trustworthiness of data | Poor decisions if data is unreliable; complex cleaning/preprocessing |
| **Value** | Oracle, 2013 | Meaningful insights and actionable benefits derived from analysis | Identifying relevant data; expertise gap; high infrastructure cost |
| **Variability** | 2015 | Inconsistency and unpredictability in data patterns over time | Adapting to sudden spikes/shifts; complex algorithms needed |
| **Visualization** | MS, 2015+ | Graphical representation of data to make insights interpretable | Showing high-dimensional data clearly; balancing detail vs. overload |

Also mentioned: **Validity** — data should be clean, accurate, reliable, valid, useful.

### "Is it Big Data?" — three-question test
1. Does the dataset exhibit one or more Vs?
2. Is it beyond the capacity of traditional RDBMS to handle?
3. Does it require specialized tools for storage, processing, or analysis?

A dataset does **not** need to exhibit all seven Vs simultaneously — one significant attribute can qualify it.

The slides include a reference table mapping example domains to the Vs:
- Social Media Analytics → all 7 Vs
- IoT Sensor Data → 6 (no Visualization marked)
- E-commerce Transactions → 6
- Healthcare Genomic Data → 6
- Real-time Stock Market → 5
- Small business accounting, personal photos, library catalogs → only 2-3 Vs (so **not** big data)

---

## 3. Applications of Big Data

### 3.1. Four types of analytics (the ladder)

| Type | Question | Based on | Example |
|---|---|---|---|
| **Descriptive** | *What happened?* | Live data, real-time ops | Sales dashboards |
| **Diagnostic** | *Why did it happen?* | Automated root-cause analysis | Why did sales drop in region X? |
| **Predictive** | *What will happen?* | Historical data, static business assumptions | Forecasting demand |
| **Prescriptive** | *What should we do?* | Optimization algorithms over forecasts | Recommend best action given goals |

### 3.2. Industry examples

- **Social Media** (Facebook ≈ 4 PB/day): user behavior, sentiment, targeted ads
- **E-Commerce**: predict preferences, optimize inventory, recommendations
- **IoT Devices**: smart homes, health monitoring, billions of real-time points
- **Healthcare**: EHRs, medical imaging, genomics → predict outbreaks, precision medicine
- **Financial Services**: stock trades, credit card txns → fraud detection, market analysis
- **Agriculture**: satellite imagery + ML for crop stress, yield improvement
- **Law enforcement**: predictive analytics (Memphis cops example shown in slides)

### 3.3. Career roles (with salary bands from slides)

- **Big Data Architect** — pipelines, storage, large-scale processing systems ($130-200k)
- **Data Engineer** — pipelines, ETL, big data platforms ($110-170k)
- **Big Data Consultant** — advisory on architectures/strategies ($120-180k)
- **Data Scientist** — analysis & predictive modeling ($100-160k)
- **AI Engineer** — AI models leveraging big data ($120-190k)

---

## 4. Annex — Data Size Units

The slides include a reference table of size units progressing from Bit → Byte → KB → MB → GB → TB → PB → EB → ZB → YB → Bronto → GeoP → Sagan → Pija → Alpha → Kryat → Amos → Pectrol → Bolger → Sambo → Quesa → Kinsa → Ruther → Dubni → Seaborg → Bohr → Hassiu → Meitner → Darmstad → Roent → Coper.

In practical terms, **PB-scale (petabyte)** is where Hadoop typically becomes necessary.

---

## Key terms (glossary)

- **Structured data** — fits a schema (tables, rows, columns).
- **Semi-structured data** — has some markers/tags (JSON, XML).
- **Unstructured data** — free-form (text, images, video).
- **RDBMS** — Relational Database Management System; the traditional incumbent that Big Data systems supersede.
- **Latency** — delay between data generation and availability of result.
- **Throughput** — volume processed per unit time.

---

## Exam targets (likely written-exam questions)

1. **Define Big Data** and list its main characteristics (the 7 Vs with one-line definitions and a challenge each).
2. **Distinguish the 4 analytics types** with an example.
3. **Given a scenario** (e.g., a hospital, a small shop, a social media platform), determine whether it's big data and which Vs apply — *justify* the answer using the 3-criterion test.
4. **Explain why one V alone** can be enough to call something big data.
5. **Discuss the role of Big Data in AI** (Big Data feeds the models; without scale, modern ML doesn't work).

### Pitfalls
- Don't confuse **Velocity** (speed of arrival) with **Volume** (size).
- **Veracity** is about *quality*, not quantity — easy to mix with Variety or Validity.
- **Value** is the *outcome*, not an intrinsic property of the data — it requires analysis.
- All 7 Vs are *not* required simultaneously — single-V datasets can qualify.
