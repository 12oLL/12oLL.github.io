---
title: "Security Wiki"
layout: single
permalink: /wiki/
author_profile: false
classes: wiki-home
---

<style>
  .terminal-container {
    max-width: 700px;
    margin: 4rem auto;
    background: #080808;
    border: 1px solid #1a1a1a;
    border-radius: 6px;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.8);
    overflow: hidden;
    font-family: 'Consolas', 'Courier New', monospace;
  }
  .terminal-header {
    background: #111;
    padding: 12px 15px;
    display: flex;
    align-items: center;
    border-bottom: 1px solid #1a1a1a;
  }
  .terminal-dots {
    display: flex;
    gap: 8px;
  }
  .dot {
    width: 12px;
    height: 12px;
    border-radius: 50%;
  }
  .dot.red { background: #ff5f56; }
  .dot.yellow { background: #ffbd2e; }
  .dot.green { background: #27c93f; }
  
  .terminal-title {
    color: #555;
    font-size: 13px;
    margin: 0 auto;
    transform: translateX(-20px); /* Keeps it perfectly centered */
  }
  .terminal-body {
    padding: 40px 30px;
    text-align: center;
  }
  .cyber-text {
    color: #06cf56;
    font-size: 2.5rem;
    margin: 0 0 15px 0;
    text-transform: uppercase;
    letter-spacing: 4px;
    text-shadow: 0 0 10px rgba(6, 207, 86, 0.3);
  }
  .terminal-prompt {
    color: #a0a0a0;
    font-size: 1rem;
    line-height: 1.6;
    margin-bottom: 30px;
    text-align: left;
    display: inline-block;
  }
  .prompt-user { color: #06cf56; }
  .prompt-dir { color: #3b8eea; }
  
  .blinking-cursor {
    display: inline-block;
    width: 8px;
    height: 18px;
    background-color: #06cf56;
    vertical-align: middle;
    animation: blink 1s step-end infinite;
    margin-left: 5px;
  }
  
  @keyframes blink {
    0%, 100% { opacity: 1; }
    50% { opacity: 0; }
  }
  
  .wiki-img {
    max-width: 100%;
    height: auto;
    border: 1px solid #222;
    border-radius: 4px;
    filter: brightness(0.8) contrast(1.2);
  }
</style>

<div class="terminal-container">
  <div class="terminal-header">
    <div class="terminal-dots">
      <div class="dot red"></div>
      <div class="dot yellow"></div>
      <div class="dot green"></div>
    </div>
    <div class="terminal-title">root@omarani:~/wiki</div>
  </div>
  
  <div class="terminal-body">
    <h2 class="cyber-text">COMING SOON</h2>
    
    <div class="terminal-prompt">
      <span class="prompt-user">omarani@system</span>:<span class="prompt-dir">~/wiki</span>$ cat status.txt <br>
      > Knowledge base initialization in progress...<br>
      > Waiting for content modules<span class="blinking-cursor"></span>
    </div>
    
    <!-- Change the src path below to your actual image -->
    <img src="/assets/images/PG.jpeg" alt="Wiki Under Construction" class="wiki-img">
  </div>
</div>