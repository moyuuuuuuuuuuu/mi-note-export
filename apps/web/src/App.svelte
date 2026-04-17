<script lang="ts">
  import type { NoteDetail } from "@core/node/typing";
  import { onMount } from "svelte";
  import NoteList from "./components/NoteList.svelte";
  import NotePreview from "./components/NotePreview.svelte";

  type SiteConfig = {
    accessPassword?: string;
  };

  let notes: NoteDetail[] = [];
  let folders: Record<string, string> = {};
  let selectedNoteId: string = "";
  let selectedNote: NoteDetail | null = null;
  let isMobileSidebarOpen = true;
  let loading = true;
  let loadError = "";

  let passwordInput = "";
  let passwordError = "";
  let accessPassword = "";
  let authKey = "mi-note-auth-public";
  let isUnlocked = true;

  onMount(async () => {
    await initApp();
  });

  $: selectedNote = notes.find((note) => note.id === selectedNoteId) || null;

  async function initApp() {
    accessPassword = await loadAccessPassword();
    authKey = createAuthKey(accessPassword);

    if (!accessPassword) {
      isUnlocked = true;
      await loadNotes();
      return;
    }

    isUnlocked = sessionStorage.getItem(authKey) === "ok";
    if (isUnlocked) {
      await loadNotes();
    }
  }

  async function loadAccessPassword() {
    try {
      const response = await fetch("/data/config.json", { cache: "no-store" });
      if (!response.ok) {
        return "";
      }
      const config: SiteConfig = await response.json();
      return (config.accessPassword || "").trim();
    } catch {
      return "";
    }
  }

  function createAuthKey(password: string) {
    if (!password) {
      return "mi-note-auth-public";
    }
    const encoded = Array.from(new TextEncoder().encode(password))
      .map((byte) => byte.toString(16).padStart(2, "0"))
      .join("")
      .slice(0, 24);
    return `mi-note-auth-${encoded}`;
  }

  async function loadNotes() {
    loading = true;
    loadError = "";
    try {
      const response = await fetch("/data/notes.json");
      if (!response.ok) {
        throw new Error(`请求失败：${response.status}`);
      }
      const data = await response.json();
      folders = data.folders || {};
      notes = data.notes || [];
    } catch (error) {
      loadError =
        error instanceof Error
          ? error.message
          : "笔记加载失败，请检查 /data/notes.json 是否存在";
    } finally {
      loading = false;
    }
  }

  async function unlockNotes() {
    if (passwordInput !== accessPassword) {
      passwordError = "密码错误，请重试。";
      return;
    }
    passwordError = "";
    sessionStorage.setItem(authKey, "ok");
    isUnlocked = true;
    await loadNotes();
  }

  function toggleMobileSidebar() {
    isMobileSidebarOpen = !isMobileSidebarOpen;
  }

  function closeMobileSidebar() {
    isMobileSidebarOpen = false;
  }

  function handleNoteSelected() {
    if (window.innerWidth <= 768) {
      closeMobileSidebar();
    }
  }
</script>

<main class="app-container">
  {#if !isUnlocked}
    <section class="lock-screen">
      <div class="lock-card">
        <h1>🔒 笔记访问保护</h1>
        <p>请输入访问密码后查看笔记内容（配置文件：/data/config.json）。</p>
        <input
          type="password"
          placeholder="请输入密码"
          bind:value={passwordInput}
          on:keydown={(event) => event.key === "Enter" && unlockNotes()}
        />
        {#if passwordError}
          <span class="error-text">{passwordError}</span>
        {/if}
        <button on:click={unlockNotes}>解锁并进入</button>
      </div>
    </section>
  {:else}
    <header class="app-header">
      <button
        class="menu-button"
        on:click={toggleMobileSidebar}
        aria-label="菜单"
      >
        <svg
          width="28"
          height="28"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <line x1="3" y1="6" x2="21" y2="6"></line>
          <line x1="3" y1="12" x2="21" y2="12"></line>
          <line x1="3" y1="18" x2="21" y2="18"></line>
        </svg>
      </button>
      <div class="header-content">
        <a
          class="header-left header-mobile"
          href="https://github.com/idootop/mi-note-export"
          target="_blank"
        >
          <div class="header-info-mobile">
            <p class="header-title" style="font-size: 16px">小米笔记备份助手</p>
            <span class="header-version">v1.0.0</span>
          </div>
        </a>
        <a
          class="header-left header-pc"
          href="https://github.com/idootop/mi-note-export"
          target="_blank"
        >
          <img src="/logo.png" alt="Logo" />
          <div class="header-info">
            <p class="header-title">小米笔记备份助手</p>
            <span class="header-version">v1.0.0</span>
          </div>
        </a>
        <div style="flex:1"></div>
        <a target="_blank" href="https://github.com/idootop" class="github-link">
          <svg
            class="github-icon"
            viewBox="0 0 256 256"
            fill="currentColor"
            stroke="currentColor"
            style="color:#111827;width:32px;height:32px"
          >
            <path
              d="M128 2.9541C57.3056 2.9541 0 60.3493 0 131.172C0 187.812 36.672 235.876 87.5392 252.823C93.9392 254 96.2688 250.045 96.2688 246.64C96.2688 243.607 96.1664 235.53 96.1024 224.842C60.4928 232.586 52.9792 207.652 52.9792 207.652C47.168 192.829 38.7712 188.887 38.7712 188.887C27.1488 180.951 39.6544 181.104 39.6544 181.104C52.4928 182 59.2512 194.314 59.2512 194.314C70.6688 213.898 89.216 208.24 96.4992 204.964C97.6768 196.682 100.979 191.037 104.64 187.837C76.224 184.599 46.336 173.591 46.336 124.464C46.336 110.474 51.328 99.0181 59.5072 90.0581C58.1888 86.8197 53.7984 73.7765 60.7616 56.1381C60.7616 56.1381 71.5136 52.6821 95.9616 69.2709C106.403 66.4231 117.177 64.9726 128 64.9573C138.827 64.9753 149.604 66.4258 160.051 69.2709C184.486 52.6821 195.213 56.1253 195.213 56.1253C202.202 73.7765 197.798 86.8197 196.493 90.0581C204.685 99.0181 209.651 110.474 209.651 124.464C209.651 173.719 179.712 184.561 151.206 187.735C155.802 191.69 159.885 199.511 159.885 211.479C159.885 228.605 159.731 242.442 159.731 246.64C159.731 250.071 162.035 254.064 168.538 252.81C194.025 244.259 216.181 227.916 231.876 206.089C247.57 184.262 256.009 158.055 256 131.172C256 60.3493 198.682 2.9541 128 2.9541Z"
            ></path>
          </svg>
        </a>
      </div>
    </header>

    <div class="main-content">
      {#if isMobileSidebarOpen}
        <div class="overlay" on:click={closeMobileSidebar}></div>
      {/if}

      <aside class="sidebar" class:open={isMobileSidebarOpen}>
        <NoteList
          {notes}
          {folders}
          bind:selectedNoteId
          on:noteSelected={handleNoteSelected}
        />
      </aside>
      <section class="content">
        {#if loading}
          <div class="status-card">正在加载笔记数据...</div>
        {:else if loadError}
          <div class="status-card error">加载失败：{loadError}</div>
        {:else}
          <NotePreview note={selectedNote} />
        {/if}
      </section>
    </div>
  {/if}
</main>

<style>
  .app-container {
    display: flex;
    flex-direction: column;
    height: 100vh;
    width: 100%;
    max-width: 1200px;
    overflow: hidden;
    background: #f8fafc;
    position: relative;
    margin: 0 auto;
    border: 1px solid #e2e8f0;
    border-radius: 20px;
    box-shadow: 0 30px 60px rgba(15, 23, 42, 0.12);
  }

  .lock-screen {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
    background: radial-gradient(circle at top, #eff6ff 0%, #f8fafc 48%, #ffffff 100%);
  }

  .lock-card {
    width: min(100%, 420px);
    background: #ffffff;
    border: 1px solid #e2e8f0;
    box-shadow: 0 20px 45px rgba(15, 23, 42, 0.08);
    border-radius: 16px;
    padding: 24px;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .lock-card h1 {
    font-size: 24px;
    color: #0f172a;
  }

  .lock-card p {
    color: #475569;
    font-size: 14px;
  }

  .lock-card input {
    border: 1px solid #cbd5e1;
    border-radius: 10px;
    padding: 11px 12px;
    font-size: 15px;
    outline: none;
    transition: all 0.2s ease;
  }

  .lock-card input:focus {
    border-color: #2563eb;
    box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.12);
  }

  .lock-card button {
    border: none;
    border-radius: 10px;
    background: linear-gradient(135deg, #2563eb, #1d4ed8);
    color: #fff;
    font-weight: 600;
    font-size: 15px;
    padding: 11px 14px;
    cursor: pointer;
    transition: transform 0.15s ease, box-shadow 0.2s ease;
  }

  .lock-card button:hover {
    transform: translateY(-1px);
    box-shadow: 0 8px 20px rgba(37, 99, 235, 0.25);
  }

  .error-text {
    color: #dc2626;
    font-size: 13px;
  }

  .app-header {
    display: flex;
    align-items: center;
    gap: 12px;
    height: 64px;
    padding: 0 20px;
    background: rgba(255, 255, 255, 0.88);
    backdrop-filter: blur(14px);
    border-bottom: 1px solid #e2e8f0;
    z-index: 30;
    flex-shrink: 0;
  }

  .menu-button {
    display: none;
  }

  .header-mobile {
    display: none !important;
  }

  .header-content {
    width: 100%;
    display: flex;
    align-items: center;
  }

  .header-left {
    display: flex;
    align-items: center;
    text-decoration: none;
    gap: 12px;
  }

  .header-left img {
    width: 40px;
    height: 40px;
  }

  .header-info {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  .header-info-mobile {
    display: flex;
    flex-direction: row;
    gap: 4px;
    align-items: center;
  }

  .header-title {
    font-size: 16px;
    font-weight: 600;
    color: #111827;
    text-decoration: none;
    display: flex;
    align-items: center;
    gap: 8px;
    transition: color 0.2s;
  }

  .header-version {
    font-size: 12px;
    color: #64748b;
    font-weight: 400;
  }

  .github-link {
    border-radius: 999px;
    padding: 6px;
    transition: background 0.2s ease;
  }

  .github-link:hover {
    background: #eff6ff;
  }

  .main-content {
    display: flex;
    flex: 1;
    overflow: hidden;
    position: relative;
  }

  .sidebar {
    width: 300px;
    flex-shrink: 0;
    background: #fff;
    z-index: 10;
    border-right: 1px solid #e2e8f0;
    transition: transform 0s ease;
  }

  .content {
    flex: 1;
    overflow: hidden;
    background: linear-gradient(180deg, rgba(248, 250, 252, 0.65) 0%, #ffffff 60%);
  }

  .status-card {
    margin: 24px;
    padding: 14px 16px;
    border-radius: 12px;
    background: #f1f5f9;
    color: #0f172a;
    border: 1px solid #e2e8f0;
  }

  .status-card.error {
    background: #fef2f2;
    border-color: #fecaca;
    color: #b91c1c;
  }

  .overlay {
    display: none;
  }

  .github-icon {
    width: 32px;
    height: 32px;
  }

  @media (max-width: 768px) {
    .app-container {
      max-width: 100%;
      border-radius: 0;
      border: none;
      box-shadow: none;
    }

    .app-header {
      height: 56px;
      padding: 0 16px;
      gap: 8px;
    }

    .github-icon {
      width: 28px !important;
      height: 28px !important;
    }

    .header-mobile {
      display: flex !important;
    }

    .header-pc {
      display: none;
    }

    .menu-button {
      display: flex;
      align-items: center;
      justify-content: center;
      border: none;
      background: transparent;
      color: #374151;
      cursor: pointer;
      border-radius: 8px;
      transition: background 0.2s;
      padding: 0;
      flex-shrink: 0;
    }

    .menu-button:active {
      background: #f3f4f6;
    }

    .header-info {
      gap: 0;
    }

    .header-title {
      font-size: 14px;
      gap: 6px;
    }

    .header-version {
      font-size: 11px;
    }

    .main-content {
      position: relative;
    }

    .sidebar {
      position: absolute;
      top: 0;
      left: 0;
      bottom: 0;
      width: 100%;
      transform: translateY(-100%);
      z-index: 20;
      border-right: none;
    }

    .sidebar.open {
      transform: translateY(0);
    }

    .overlay {
      display: block;
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      background: rgba(15, 23, 42, 0.45);
      z-index: 15;
      animation: fadeIn 0.25s ease;
    }

    @keyframes fadeIn {
      from {
        opacity: 0;
      }
      to {
        opacity: 1;
      }
    }

    .content {
      width: 100%;
    }
  }
</style>
