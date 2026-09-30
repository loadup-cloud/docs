---
title: "LoadUp Framework"
layout: hextra-home
toc: false
---

<div class="lu-home">
  <section class="lu-hero" aria-labelledby="lu-hero-title">
    <div class="lu-hero-copy">
      <p class="lu-eyebrow"><span class="lu-status-dot"></span> LOADUP FRAMEWORK <span class="lu-eyebrow-divider">/</span> SPRING BOOT SDK</p>
      <h1 id="lu-hero-title">基础能力按需组合，<br><span>业务开发保持专注。</span></h1>
      <p class="lu-lead">LoadUp 为 Spring Boot 应用提供可复用的通用基础、技术组件和业务模块。通过 BOM 统一版本，只引入当前需要的能力。</p>
      <div class="lu-actions">
        <a class="lu-button lu-button-primary" href="docs/">开始阅读 <span aria-hidden="true">↗</span></a>
        <a class="lu-button lu-button-secondary" href="docs/components/">浏览组件 <span aria-hidden="true">→</span></a>
      </div>
      <p class="lu-hero-note">面向集成方开发者 · 基于 Spring Boot · 按需引入</p>
    </div>
    <div class="lu-hero-visual" aria-label="LoadUp 组件组织示意">
      <div class="lu-visual-top"><span class="lu-visual-dots" aria-hidden="true"><i></i><i></i><i></i></span><span>your-spring-boot-app</span><span class="lu-visual-ready">READY</span></div>
      <div class="lu-visual-body">
        <div class="lu-visual-heading"><span class="lu-visual-symbol">{ }</span><div><strong>按需装配</strong><small>从 BOM 到业务能力</small></div></div>
        <div class="lu-visual-row"><span class="lu-row-index">01</span><span class="lu-row-icon">◆</span><span><strong>loadup-dependencies</strong><small>统一依赖版本</small></span><span class="lu-row-check">✓</span></div>
        <div class="lu-visual-row"><span class="lu-row-index">02</span><span class="lu-row-icon">◈</span><span><strong>components / *</strong><small>缓存、认证、数据库等能力</small></span><span class="lu-row-check">✓</span></div>
        <div class="lu-visual-row"><span class="lu-row-index">03</span><span class="lu-row-icon">▣</span><span><strong>modules / *</strong><small>UPMS 等通用业务模块</small></span><span class="lu-row-check">✓</span></div>
        <div class="lu-visual-foot"><span class="lu-terminal-prompt">$</span> 选择能力，接入你的应用<span class="lu-cursor" aria-hidden="true"></span></div>
      </div>
    </div>
  </section>

  <div class="lu-principles" aria-label="框架特点"><span>统一版本管理</span><span>按需引入</span><span>遵循 Spring 生态约定</span><span>业务模块可复用</span></div>

  <section class="lu-section" aria-labelledby="lu-start-title">
    <div class="lu-section-head"><p class="lu-kicker">START HERE <span>01 / 03</span></p><h2 id="lu-start-title">从你正在做的事开始</h2><p>不用先读完全部文档。选一条路径，直接进入对应的接入说明。</p></div>
    <div class="lu-path-grid">
      <a class="lu-path-card" href="docs/dependencies/"><span class="lu-card-number">01 / 接入</span><span class="lu-card-glyph">&lt;/&gt;</span><h3>引入 BOM</h3><p>统一管理 LoadUp 模块版本，开始在自己的 Spring Boot 项目中组合能力。</p><span class="lu-card-link">查看依赖管理 <span aria-hidden="true">↗</span></span></a>
      <a class="lu-path-card" href="docs/components/"><span class="lu-card-number">02 / 选型</span><span class="lu-card-glyph">▦</span><h3>挑选技术组件</h3><p>从缓存、数据库、认证、文件存储等能力中选择适合当前项目的实现。</p><span class="lu-card-link">探索技术组件 <span aria-hidden="true">↗</span></span></a>
      <a class="lu-path-card" href="docs/modules/"><span class="lu-card-number">03 / 扩展</span><span class="lu-card-glyph">◎</span><h3>接入业务模块</h3><p>使用 UPMS 等通用业务能力，将精力放在自己的领域逻辑上。</p><span class="lu-card-link">浏览业务模块 <span aria-hidden="true">↗</span></span></a>
    </div>
  </section>

  <section class="lu-section lu-section-capabilities" aria-labelledby="lu-capabilities-title">
    <div class="lu-section-head"><p class="lu-kicker">THE SYSTEM <span>02 / 03</span></p><h2 id="lu-capabilities-title">清晰分层，自由组合</h2><p>基础约定、技术集成和通用业务各有边界；测试与本地验证也有独立入口。</p></div>
    <div class="lu-cap-grid">
      <a href="docs/commons/" class="lu-cap-card"><span class="lu-cap-mark">C</span><span><strong>Commons</strong><small>DTO、工具、日志与链路追踪</small></span><span class="lu-cap-arrow" aria-hidden="true">↗</span></a>
      <a href="docs/components/" class="lu-cap-card"><span class="lu-cap-mark">T</span><span><strong>Components</strong><small>按需接入的技术能力</small></span><span class="lu-cap-arrow" aria-hidden="true">↗</span></a>
      <a href="docs/modules/" class="lu-cap-card"><span class="lu-cap-mark">M</span><span><strong>Modules</strong><small>可复用的通用业务模块</small></span><span class="lu-cap-arrow" aria-hidden="true">↗</span></a>
      <a href="docs/testing/" class="lu-cap-card"><span class="lu-cap-mark">V</span><span><strong>Testify</strong><small>集成测试与能力验证</small></span><span class="lu-cap-arrow" aria-hidden="true">↗</span></a>
    </div>
  </section>

  <section class="lu-bottom" aria-labelledby="lu-bottom-title"><div><p class="lu-kicker">READY TO BUILD <span>03 / 03</span></p><h2 id="lu-bottom-title">找到需要的能力，然后开始构建。</h2><p>每个组件页面将接入说明与设计细节放在一起，方便从使用继续深入。</p></div><a class="lu-button lu-button-light" href="docs/">进入文档 <span aria-hidden="true">↗</span></a></section>
</div>
