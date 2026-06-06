---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

<style>
  .about-wrapper { max-width: 640px; margin: 0 auto; padding: 1rem 0 3rem; }

  .about-profile { text-align: center; margin-bottom: 2.5rem; }
  .about-profile .name { font-size: 2rem; font-weight: 700; margin: 0 0 0.4rem; }
  .about-profile .tagline { font-size: 1rem; color: var(--text-muted-color); margin: 0 0 1.2rem; }
  .about-profile .divider {
    width: 40px; height: 3px;
    background: var(--link-color, #5a9cf5);
    margin: 0 auto;
    border-radius: 2px;
  }

  .about-section { margin-bottom: 2rem; }
  .about-section-title {
    font-size: 0.75rem;
    font-weight: 600;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: var(--text-muted-color);
    margin-bottom: 0.75rem;
  }
  .about-section p {
    font-size: 0.95rem;
    line-height: 1.8;
    color: var(--text-color);
    margin: 0;
  }

  .badge-group { display: flex; flex-wrap: wrap; gap: 0.5rem; }
  .badge {
    display: inline-flex; align-items: center; gap: 0.35rem;
    padding: 0.3rem 0.75rem;
    border-radius: 999px;
    border: 1px solid var(--border-color, #3a3a3a);
    font-size: 0.82rem;
    color: var(--text-color);
  }
  .badge i { font-size: 0.75rem; color: var(--link-color, #5a9cf5); }

  .career-item {
    display: flex; gap: 1rem; align-items: flex-start;
    padding: 0.75rem 0;
    border-bottom: 1px solid var(--border-color, #2a2a2a);
  }
  .career-item:last-child { border-bottom: none; }
  .career-period {
    font-size: 0.78rem;
    color: var(--text-muted-color);
    white-space: nowrap;
    padding-top: 0.1rem;
    min-width: 90px;
  }
  .career-content .role { font-size: 0.92rem; font-weight: 600; margin-bottom: 0.15rem; }
  .career-content .desc { font-size: 0.82rem; color: var(--text-muted-color); }
</style>

<div class="about-wrapper">

  <div class="about-profile">
    <div class="name">신희성</div>
    <div class="tagline">Game Programmer · Unreal Engine · C++</div>
    <div class="divider"></div>
  </div>

  <div class="about-section">
    <div class="about-section-title">About</div>
    <p>게임 프로그래머를 목표로 공부하고 있습니다.<br>
    Unreal Engine 5와 C++를 중심으로 게임 시스템 구현과 문제 해결 과정을 기록합니다.</p>
  </div>

  <div class="about-section">
    <div class="about-section-title">Skills</div>
    <div class="badge-group">
      <span class="badge"><i class="fas fa-gamepad"></i> Unreal Engine 5</span>
      <span class="badge"><i class="fas fa-code"></i> C++</span>
      <span class="badge"><i class="fas fa-code"></i> Blueprint</span>
      <span class="badge"><i class="fas fa-network-wired"></i> GAS</span>
      <span class="badge"><i class="fas fa-code-branch"></i> Git</span>
      <span class="badge"><i class="fas fa-robot"></i> RPA</span>
    </div>
  </div>

  <div class="about-section">
    <div class="about-section-title">Career</div>
    <div class="career-item">
      <div class="career-period">2019 — 2024</div>
      <div class="career-content">
        <div class="role">RPA 개발자</div>
        <div class="desc">업무 자동화 시스템 개발 · 협업 · 유지보수</div>
      </div>
    </div>
    <div class="career-item">
      <div class="career-period">2024 — 현재</div>
      <div class="career-content">
        <div class="role">게임 프로그래머 전향</div>
        <div class="desc">Unreal Engine 5 · C++ · 게임 시스템 구현</div>
      </div>
    </div>
  </div>

  <div class="about-section">
    <div class="about-section-title">This Blog</div>
    <div class="badge-group">
      <span class="badge"><i class="fas fa-gamepad"></i> Unreal Engine</span>
      <span class="badge"><i class="fas fa-code"></i> C++</span>
      <span class="badge"><i class="fas fa-pencil-alt"></i> 코딩테스트</span>
      <span class="badge"><i class="fas fa-code-branch"></i> Git</span>
      <span class="badge"><i class="fas fa-bug"></i> 트러블슈팅</span>
    </div>
  </div>

</div>
