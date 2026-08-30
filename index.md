---
layout: default
title: Home
---

<article class="hero">
  <p class="eyebrow">AI systems &amp; engineering</p>
  <h1>{{ site.title }}</h1>
  <p class="tagline">{{ site.description }}</p>
</article>

<article class="content-page">
  <section class="section" id="contents">
    <h2>Table of contents</h2>
    <ol class="toc">
      <li><a href="#bio">Bio</a></li>
      <li><a href="#expertise">Core expertise</a></li>
      <li><a href="#writing">Writing</a></li>
    </ol>
  </section>

  <section class="section" id="bio">
    <h2>Bio</h2>
    <p>Axoryq AI Innovations designs practical, resilient AI systems for organizations operating at real-world scale. Our work connects modern generative AI with the data platforms, software engineering, and operational rigor needed to make it dependable.</p>
    <p>We focus on turning promising AI capabilities into useful products and platforms: systems that are measurable, secure, maintainable, and designed around the people who use them.</p>
  </section>

  <section class="section" id="expertise">
    <h2>Core expertise</h2>
    <ul class="expertise-list">
      <li><strong>Agentic AI &amp; orchestration</strong>Multi-agent systems, planner/executor and supervisor patterns, tool routing, memory, workflow orchestration, function calling, MCP, and A2A.</li>
      <li><strong>Generative AI &amp; LLM systems</strong>RAG, custom retrieval workflows, in-context learning, model fine-tuning, evaluations, and feedback loops across quality, safety, cost, and latency.</li>
      <li><strong>AI/ML engineering</strong>Machine learning, forecasting, anomaly detection, classification, regression, log-pattern recognition, adaptive learning, world models, and digital twins.</li>
      <li><strong>Data &amp; streaming platforms</strong>Lakehouse architectures, Spark, Kafka, Flink, Iceberg, Delta Lake, and reactive systems for high-volume data streams.</li>
      <li><strong>Infrastructure &amp; software engineering</strong>Vector and graph infrastructure, Kubernetes, observability, distributed systems, microservices, and production APIs.</li>
      <li><strong>Industrial &amp; enterprise AI</strong>Factory automation, predictive analytics, AIOps, event-driven architectures, equipment health, and time-series intelligence.</li>
      <li><strong>Technical leadership</strong>Enterprise architecture, platform strategy, mentoring, governance, security, reference architectures, and roadmaps.</li>
    </ul>
  </section>

  <section class="section" id="writing">
    <h2>Writing</h2>
    {% if site.posts.size > 0 %}
    <ul class="post-list">
      {% for post in site.posts %}
      <li>
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
        <time class="post-date" datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d, %Y" }}</time>
      </li>
      {% endfor %}
    </ul>
    {% else %}
    <p>New writing will appear here.</p>
    {% endif %}
  </section>
</article>
