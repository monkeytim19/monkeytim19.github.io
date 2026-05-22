---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

{% include base_path %}

Experience
======

* **Data Scientist**, HSBC – London, UK (Jun 2025 – Present)
  * Performed regular feature engineering on Google Cloud Platform (GCP) to continually improve production LightGBM models for detecting payment fraud
  * Implemented novel post-training transformations to score predictions and applied ensembling of multiple specialised ML models to improve overall value detection rate
  * Improved optimisation procedure on model thresholding to improve monthly value detected by 3–5 percentage points
  * Refined the SAFE model and achieved competitive KPIs to the LightGBM whilst detecting 30–40% distinct fraudulent cases and yielding over 5 to 10% additional annual savings
  * Collaborated with cross-functional team following the Agile methodology and standard Github practices to maintain data quality and explore model improvement ideas

* **AI Engineer**, HSBC – Birmingham, UK (Oct 2024 – Jun 2025)
  * Engineered novel deep learning GPT-like model for fraud detection (SAFE) using HuggingFace/PyTorch
  * Optimised scripts for preprocessing event logs and payments dataset with over 10 billion rows on GCP
  * Using distributed data parallel with multiple GPUs, pre-trained the model from scratch with next-token prediction on tabular data and fine-tuned it with parameter efficient techniques like low-rank adaptation (LoRA)
  * Researched and experimented with methodologies from academic papers on topics related to model architecture, tokenisation, time encodings for improving model performance
  * Engaged in use-cases involving prompt engineering and the orchestration of Large Language Models for purposes like automated feature engineering and conversational chatbot

* **Research Assistant**, The Alan Turing Institute – London, UK (Jun 2024 – Sep 2024)
  * Performed an empirical analysis on the existing physics-informed trajectory prediction model as part of Project Bluebird
  * Identified the shortcomings of the prediction model by corroborating results on a large-scale dataset of over 100,000 UK flights across 3 months

* **Research Assistant**, Imperial College London – London, UK (Jan 2022 – Jun 2022)
  * Assisted Dr. Andre Veiga on the summarisation of content and teaching material related to the economics of online platforms and Game Theory

Education
======

* **MSc in Artificial Intelligence**, Imperial College London (Oct 2023 – Sep 2024)
  * Individual research project supervised by Dr. Pedro Mediano – *Repairing Proofs in Lean with AI: A Pipeline for Fine-Tuning and Evaluating Large Language Models in Theorem Proving*

* **Undergraduate courses in Mathematics, Computer Science, Philosophy**, University of Toronto (St. George) (Sep 2022 – May 2023)

* **MSc in Econometrics and Mathematical Economics & BSc in Economics**, London School of Economics & Political Science (Sep 2018 – Jul 2022)

Publications
======

  <ul>{% for post in site.publications reversed %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>

Projects
======

  <ul>{% for post in site.portfolio reversed %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>

Skills
======

* **Programming Languages:** Python (PyTorch, Scikit-learn, HuggingFace, PySpark, etc.), SQL, R, MATLAB, Shell (Bash/Zsh)
* **Technologies:** Google Cloud Platform, LaTeX, GitHub, Apache Spark
* **Languages:** English, Cantonese, Mandarin
