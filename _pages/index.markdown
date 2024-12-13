---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: splash
title: OddDotNet
permalink: /
# header:
#   overlay_text: OddDotNet
# excerpt: "OpenTelemetry test harness for any language, built in .NET"
intro:
  - 
    excerpt: "OpenTelemetry test harness for any language, built in .NET"
    title: "Welcome to OddDotNet"
feature_row:
  - 
    image_path: /assets/images/OpenTelemetry.png
    alt: "OpenTelemetry placeholder"
    title: "OpenTelemetry"
    excerpt: "Verify OpenTelemetry Signal Generation"
    url: "quick-starts/csharp/aspire/"
    btn_label: "Read More"
    btn_class: "btn--primary"
feature_row2:
  - 
    image_path: /assets/images/OtelCollector.png
    alt: "OTel Collector placeholder"
    title: "OTel Collector"
    excerpt: "Easily Validate Otel Collector Configuration"
    url: "quick-starts/csharp/otelcol/"
    btn_label: "Read More"
    btn_class: "btn--primary"
feature_row3:
  - 
    image_path: /assets/images/427111.png
    alt: "Load Test placeholder"
    title: "Load Test"
    excerpt: "Load Test Using Real Production Signals"
    url: "coming-soon"
    btn_label: "Coming Soon"
    btn_class: "btn--disabled"
---

{% include feature_row id="intro" type="center" %}

{% include feature_row type="center" %}

{% include feature_row id="feature_row2" type="center" %}

{% include feature_row id="feature_row3" type="center" %}