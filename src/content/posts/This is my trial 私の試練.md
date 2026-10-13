---
title: This is my trial 私の試練（我的考驗）— 山口百惠
published: 2026-10-13
description: 山口百惠經典歌曲「This is my trial 私の試練」，附中日歌詞與 YouTube 同步高亮播放。
tags: [山口百惠, Momoe Yamaguchi, 谷村新司, J-POP, 經典歌曲, 歌詞同步]
category: Music
draft: false
---

# This is my trial 私の試練（我的考驗）

**歌手**：山口百惠（Momoe Yamaguchi）  
**作詞／作曲**：谷村新司  
**發行**：1980年10月21日

<div id="player-container" style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%;">
  <div id="yt-player"></div>
</div>

<div id="lyrics-box" style="margin-top: 24px; max-height: 420px; overflow-y: auto; border: 1px solid #e0e0e0; border-radius: 8px; padding: 16px; background: #fafafa; font-size: 1.05rem; line-height: 1.7;">
  <!-- 歌詞會由 JS 動態生成 -->
</div>

<style>
  .lyric-line {
    padding: 6px 10px;
    margin: 2px 0;
    border-radius: 6px;
    transition: all 0.25s ease;
    cursor: pointer;
  }
  .lyric-line:hover {
    background: #f0f0f0;
  }
  .lyric-line.active {
    background: #ffe082;
    color: #000;
    font-weight: 600;
    transform: scale(1.02);
  }
  .lyric-line .time {
    display: inline-block;
    min-width: 48px;
    color: #888;
    font-size: 0.85em;
    margin-right: 8px;
  }
</style>

<script>
  // ========== 歌詞時間戳（秒）==========
  // 根據公開 LRC 資料整理，可能與影片略有偏差，可自行微調
  const lyricsData = [
    { time: 15.72, text: "This is my trial 濡れた歩道を" },
    { time: 20.91, text: "It's lonesome trial ただひたすらに" },
    { time: 33.75, text: "ひきかえせない ふりむきもしない" },
    { time: 54.67, text: "そう私は今まぎれもなく 自分で歩きはじめる" },
    { time: 68.54, text: "This is my trial 私のゴールは" },
    { time: 81.88, text: "数えきれない人達の胸じゃない" },

    { time: 103.49, text: "This is my trial 仮面を捨てて" },
    { time: 111.49, text: "It's lonesome trial 素顔で雨に向えば" },
    { time: 122.75, text: "ほほを伝わる あつい涙よ" },
    { time: 142.31, text: "そう私は今まぎれもなく 自分で歩きはじめる" },
    { time: 157.05, text: "This is my trial 私のゴールは" },
    { time: 170.56, text: "数えきれない人達の胸じゃない" },

    { time: 209.62, text: "This is my trial 濡れた歩道を" },
    { time: 215.67, text: "It's lonesome trial ただひたすらに" },
    { time: 227.24, text: "ひきかえせない ふりむきもしない" },
    { time: 246.65, text: "そう私は今まぎれもなく 歩き歩きはじめる" },
    { time: 261.95, text: "This is my trial 私のゴールは" },
    { time: 274.35, text: "数えきれない人達の胸じゃない" },

    { time: 287.66, text: "This is my trial 私のゴールは" },
    { time: 300.30, text: "数えきれない人達の胸じゃない" }
  ];

  let player;
  let currentIndex = -1;
  let pollInterval = null;

  // 載入 YouTube IFrame API
  function loadYouTubeAPI() {
    if (window.YT && window.YT.Player) {
      onYouTubeIframeAPIReady();
      return;
    }
    const tag = document.createElement('script');
    tag.src = "https://www.youtube.com/iframe_api";
    const firstScriptTag = document.getElementsByTagName('script')[0];
    firstScriptTag.parentNode.insertBefore(tag, firstScriptTag);
  }

  // YouTube API 準備好後建立播放器
  window.onYouTubeIframeAPIReady = function() {
    player = new YT.Player('yt-player', {
      height: '100%',
      width: '100%',
      videoId: 'T9s8LBNs95M',
      playerVars: {
        'playsinline': 1,
        'rel': 0,
        'modestbranding': 1
      },
      events: {
        'onReady': onPlayerReady,
        'onStateChange': onPlayerStateChange
      }
    });

    // 讓播放器填滿容器
    const container = document.getElementById('player-container');
    const iframe = document.getElementById('yt-player');
    if (iframe) {
      iframe.style.position = 'absolute';
      iframe.style.top = '0';
      iframe.style.left = '0';
      iframe.style.width = '100%';
      iframe.style.height = '100%';
    }
  };

  function onPlayerReady(event) {
    renderLyrics();
  }

  function onPlayerStateChange(event) {
    if (event.data === YT.PlayerState.PLAYING) {
      startPolling();
    } else {
      stopPolling();
    }
  }

  function startPolling() {
    stopPolling();
    pollInterval = setInterval(updateLyrics, 200); // 每 200ms 檢查一次
  }

  function stopPolling() {
    if (pollInterval) {
      clearInterval(pollInterval);
      pollInterval = null;
    }
  }

  function updateLyrics() {
    if (!player || typeof player.getCurrentTime !== 'function') return;
    const currentTime = player.getCurrentTime();

    let newIndex = -1;
    for (let i = 0; i < lyricsData.length; i++) {
      if (currentTime >= lyricsData[i].time) {
        newIndex = i;
      } else {
        break;
      }
    }

    if (newIndex !== currentIndex && newIndex >= 0) {
      // 移除舊高亮
      const oldActive = document.querySelector('.lyric-line.active');
      if (oldActive) oldActive.classList.remove('active');

      // 加上新高亮並捲動
      const lines = document.querySelectorAll('.lyric-line');
      if (lines[newIndex]) {
        lines[newIndex].classList.add('active');
        lines[newIndex].scrollIntoView({ behavior: 'smooth', block: 'center' });
      }
      currentIndex = newIndex;
    }
  }

  function renderLyrics() {
    const box = document.getElementById('lyrics-box');
    box.innerHTML = '';

    lyricsData.forEach((item, index) => {
      const div = document.createElement('div');
      div.className = 'lyric-line';
      div.dataset.index = index;

      const minutes = Math.floor(item.time / 60);
      const seconds = Math.floor(item.time % 60).toString().padStart(2, '0');
      div.innerHTML = `<span class="time">\( {minutes}: \){seconds}</span>${item.text}`;

      // 點擊歌詞可跳轉到該時間
      div.addEventListener('click', () => {
        if (player && player.seekTo) {
          player.seekTo(item.time, true);
          player.playVideo();
        }
      });

      box.appendChild(div);
    });
  }

  // 啟動
  loadYouTubeAPI();
</script>

---
