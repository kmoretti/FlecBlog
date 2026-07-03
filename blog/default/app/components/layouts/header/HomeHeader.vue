<script lang="ts" setup>
const { basicConfig } = useSysConfig();
const { themeConfig } = useTheme();
const displayText = ref('');
const typingSpeed = 150;
const deletingSpeed = 80;
const pauseTime = 2000;
let typingTimer: number | null = null;
let isInitialized = false;

const getTypingTexts = (): string[] => {
  try {
    const raw = themeConfig.value.typing_texts;
    if (!raw) return [];
    const parsed = typeof raw === 'string' ? JSON.parse(raw) : raw;
    if (!Array.isArray(parsed)) return [];
    return parsed.map((item: unknown) => {
      if (typeof item === 'string') return item;
      if (item && typeof item === 'object' && 'value' in (item as Record<string, unknown>)) {
        return String((item as Record<string, string>).value);
      }
      return '';
    }).filter(Boolean);
  } catch {
    return [];
  }
};

const scrollToContent = () => {
  window.scrollTo({
    top: window.innerHeight - 64,
    behavior: 'smooth',
  });
};

const startTypeWriter = () => {
  const texts = getTypingTexts();
  if (texts.length === 0) return;

  if (typingTimer) {
    clearTimeout(typingTimer);
  }
  displayText.value = '';

  let textIndex = 0;
  let charIndex = 0;
  let isDeleting = false;

  const animate = () => {
    const currentText = texts[textIndex];
    if (!currentText) return;

    if (!isDeleting) {
      if (charIndex < currentText.length) {
        displayText.value += currentText.charAt(charIndex);
        charIndex++;
        typingTimer = window.setTimeout(animate, typingSpeed);
      } else {
        isDeleting = true;
        typingTimer = window.setTimeout(animate, pauseTime);
      }
    } else {
      if (charIndex > 0) {
        displayText.value = currentText.substring(0, charIndex - 1);
        charIndex--;
        typingTimer = window.setTimeout(animate, deletingSpeed);
      } else {
        isDeleting = false;
        textIndex = (textIndex + 1) % texts.length;
        typingTimer = window.setTimeout(animate, typingSpeed);
      }
    }
  };

  animate();
};

watch(
  () => themeConfig.value.typing_texts,
  (val) => {
    if (val) {
      isInitialized = true;
      startTypeWriter();
    }
  },
);

onMounted(() => {
  if (getTypingTexts().length > 0) {
    isInitialized = true;
    setTimeout(startTypeWriter, 500);
  }
});

onUnmounted(() => {
  if (typingTimer) {
    clearTimeout(typingTimer);
  }
});
</script>

<template>
  <header class="home-header">
    <div class="site-info">
      <h1>{{ basicConfig.title }}</h1>
      <div class="site-subtitle">
        <span id="subtitle">{{ displayText }}</span>
        <span class="cursor">|</span>
      </div>
    </div>
    <div class="scroll-indicator" @click="scrollToContent">
      <i class="ri-arrow-down-s-line ri-2x" />
    </div>
  </header>
</template>

<style lang="scss" scoped>
.home-header {
  position: relative;
  height: calc(100vh - 4rem);
  width: 100%;

  .site-info {
    position: absolute;
    top: 35%;
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-direction: column;

    h1 {
      font-size: 2.6rem;
      color: #fff;
    }

    .site-subtitle {
      font-size: 1.7rem;
      color: #eee;

      .cursor {
        display: inline-block;
        margin-left: 4px;
        animation: blink 1s infinite;
      }
    }
  }

  .scroll-indicator {
    position: absolute;
    bottom: 10px;
    width: 100%;
    animation: bounce 1.5s infinite;
    cursor: pointer;

    i {
      color: #eee;
      position: relative;
      text-align: center;
      width: 100%;
    }
  }
}

@keyframes bounce {
  0% {
    opacity: 0.4;
    transform: translate(0, 0);
  }

  50% {
    opacity: 1;
    transform: translate(0, -16px);
  }

  100% {
    opacity: 0.4;
    transform: translate(0, 0);
  }
}

@keyframes blink {
  0%,
  49% {
    opacity: 1;
  }

  50%,
  100% {
    opacity: 0;
  }
}

// 响应式设计
@media screen and (max-width: 768px) {
  .home-header {
    .site-info {
      h1 {
        font-size: 2.2rem;
      }

      .site-subtitle {
        font-size: 1.4rem;
      }
    }
  }
}
</style>
