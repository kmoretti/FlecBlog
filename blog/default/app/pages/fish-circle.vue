<script setup lang="ts">
interface FriendArticle {
  title: string;
  created: string | number;
  updated: string | number;
  link: string;
  author: string;
  avatar: string;
  id?: number;
}

interface FriendCircleResponse {
  article_data?: FriendArticle[];
  statistical_data?: {
    article_num?: number;
    friends_num?: number;
    last_updated_time?: string;
  };
}

definePageMeta({
  showSidebar: false,
});

useSeoMeta({
  title: '鱼塘',
  description: '友链朋友圈 - 浏览朋友们的最新文章',
});

const toTs = (v: string | number | undefined): number => {
  if (!v) return 0;
  if (typeof v === 'number') return v;
  const d = new Date(v);
  return d.getTime() / 1000;
};

const { themeConfig } = useTheme();
const fetchUrl = computed(() => themeConfig.value.friend_circle_api_url || '');

const {
  data: articles,
  pending,
  error,
  refresh,
} = useAsyncData(
  'fish-circle',
  async () => {
    const url = fetchUrl.value;
    if (!url) return [];
    const res = await $fetch<FriendArticle[] | FriendCircleResponse>(url);
    const list = Array.isArray(res) ? res : (res.article_data ?? []);
    return list
      .filter(a => a.title && a.link)
      .sort((a, b) => toTs(b.updated || b.created) - toTs(a.updated || a.created));
  },
  {
    watch: [fetchUrl],
  }
);

const stats = computed(() => {
  const list = articles.value ?? [];
  const uniqueAuthors = new Set(list.map(a => a.author));
  const latest = list[0];
  return {
    total: list.length,
    authors: uniqueAuthors.size,
    latestDate: latest ? latest.updated || latest.created : null,
  };
});

const randomArticle = ref<FriendArticle | null>(null);

const pickRandom = () => {
  const list = articles.value ?? [];
  if (list.length === 0) return;
  randomArticle.value = list[Math.floor(Math.random() * list.length)] ?? null;
};

watch(articles, () => pickRandom(), { once: true });

const formatDate = (v: string | number | undefined): string => {
  if (!v) return '';
  const ts = typeof v === 'number' ? v * 1000 : new Date(v).getTime();
  const d = new Date(ts);
  const y = d.getFullYear();
  const m = String(d.getMonth() + 1).padStart(2, '0');
  const day = String(d.getDate()).padStart(2, '0');
  return `${y}-${m}-${day}`;
};

const timeAgo = (v: string | number | undefined): string => {
  if (!v) return '';
  const ts = typeof v === 'number' ? v * 1000 : new Date(v).getTime();
  const now = Date.now();
  const diff = now - ts;
  const minutes = Math.floor(diff / 60000);
  if (minutes < 1) return '刚刚';
  if (minutes < 60) return `${minutes}分钟前`;
  const hours = Math.floor(minutes / 60);
  if (hours < 24) return `${hours}小时前`;
  const days = Math.floor(hours / 24);
  if (days < 30) return `${days}天前`;
  const months = Math.floor(days / 30);
  return `${months}个月前`;
};

const authorFilter = ref('');

const filteredArticles = computed(() => {
  const list = articles.value ?? [];
  if (!authorFilter.value) return list;
  return list.filter(a => a.author.includes(authorFilter.value));
});

const uniqueAuthors = computed(() => {
  const list = articles.value ?? [];
  return [...new Set(list.map(a => a.author))].sort();
});

const showAllAuthors = ref(false);
const authorPreviewLimit = 12;
const displayedAuthors = computed(() => {
  if (showAllAuthors.value) return uniqueAuthors.value;
  return uniqueAuthors.value.slice(0, authorPreviewLimit);
});
const hasMoreAuthors = computed(() => uniqueAuthors.value.length > authorPreviewLimit);

const isEmpty = computed(() => !fetchUrl.value);
</script>

<template>
  <div id="fish-circle-page">
    <div class="fish-header">
      <div class="header-content">
        <h1 class="page-title">鱼塘</h1>
        <p class="page-desc">泛览朋友们的最近文字，垂钓有趣的思想</p>
      </div>
    </div>

    <!-- 空状态：未配置 API -->
    <div v-if="isEmpty" class="empty-config">
      <span class="empty-icon">🐟</span>
      <p>请在后台「主题配置」→「页面配置」中设置鱼塘API地址</p>
    </div>

    <template v-else>
      <!-- 加载状态 -->
      <div v-if="pending" class="loading-status">
        <div class="loading-spinner" />
        <span>正在撒网...</span>
      </div>

      <!-- 错误状态 -->
      <div v-else-if="error" class="error-status">
        <span class="error-icon">⚠️</span>
        <p>获取鱼塘数据失败，请检查API地址配置</p>
        <button class="retry-btn" @click="() => refresh()">重试</button>
      </div>

      <template v-else-if="articles?.length">
        <!-- 统计面板 -->
        <div class="stats-panel">
          <div class="stat-card">
            <span class="stat-value">{{ stats.total }}</span>
            <span class="stat-label">文章</span>
          </div>
          <div class="stat-card">
            <span class="stat-value">{{ stats.authors }}</span>
            <span class="stat-label">作者</span>
          </div>
          <div class="stat-card">
            <span class="stat-value">
              {{ stats.latestDate ? timeAgo(stats.latestDate) : '暂无' }}
            </span>
            <span class="stat-label">最近更新</span>
          </div>
          <div class="stat-card" @click="pickRandom">
            <span class="stat-value">🎲</span>
            <span class="stat-label">随机一文</span>
          </div>
        </div>

        <!-- 随机文章 -->
        <div v-if="randomArticle" class="random-article">
          <div class="random-meta">
            <NuxtImg
              v-if="randomArticle.avatar"
              :src="randomArticle.avatar"
              :alt="randomArticle.author"
              class="random-avatar"
              loading="lazy"
            />
            <span class="random-author">{{ randomArticle.author }}</span>
            <span class="random-date">
              {{ formatDate(randomArticle.updated || randomArticle.created) }}
            </span>
          </div>
          <a
            :href="randomArticle.link"
            target="_blank"
            rel="noopener noreferrer"
            class="random-title"
          >
            {{ randomArticle.title }}
          </a>
          <button class="random-btn" @click="pickRandom">
            <span>换一篇</span>
          </button>
        </div>

        <!-- 搜索与筛选 -->
        <div class="filter-bar">
          <div class="search-wrap">
            <i class="ri-search-line" />
            <input
              v-model="authorFilter"
              type="text"
              class="search-input"
              placeholder="搜索作者..."
            />
          </div>
          <div class="author-chips">
            <button :class="['chip', { active: !authorFilter }]" @click="authorFilter = ''">
              全部
            </button>
            <button
              v-for="author in displayedAuthors"
              :key="author"
              :class="['chip', { active: authorFilter === author }]"
              @click="authorFilter = authorFilter === author ? '' : author"
            >
              {{ author }}
            </button>
            <button
              v-if="hasMoreAuthors"
              class="chip chip-more"
              @click="showAllAuthors = !showAllAuthors"
            >
              {{ showAllAuthors ? '收起' : `全部 ${uniqueAuthors.length} 位` }}
            </button>
          </div>
        </div>

        <!-- 文章网格 -->
        <div class="article-grid">
          <a
            v-for="article in filteredArticles"
            :key="article.link + article.author"
            :href="article.link"
            target="_blank"
            rel="noopener noreferrer"
            class="article-card"
            :title="article.title"
          >
            <div class="card-body">
              <h3 class="card-title">{{ article.title }}</h3>
              <div class="card-meta">
                <div class="card-author">
                  <NuxtImg
                    v-if="article.avatar"
                    :src="article.avatar"
                    :alt="article.author"
                    class="author-avatar"
                    loading="lazy"
                    referrerpolicy="no-referrer"
                  />
                  <span class="author-name">{{ article.author }}</span>
                </div>
                <span class="card-date">{{ formatDate(article.updated || article.created) }}</span>
              </div>
            </div>
          </a>
        </div>

        <!-- 空筛选结果 -->
        <div v-if="filteredArticles.length === 0 && !pending" class="no-results">
          <p>没有找到匹配的文章</p>
        </div>
      </template>

      <!-- 无数据 -->
      <div v-else class="no-data">
        <span class="empty-icon">🐟</span>
        <p>鱼塘暂时还是空的</p>
      </div>
    </template>
  </div>
</template>

<style lang="scss" scoped>
@use '@/assets/css/mixins' as *;

#fish-circle-page {
  width: 100%;
}

.fish-header {
  margin-bottom: 28px;
  padding: 40px;
  @extend .cardHover;

  .header-content {
    text-align: center;

    .page-title {
      margin: 0 0 8px;
      font-size: 2rem;
      font-weight: bold;
      line-height: 1.2;
    }

    .page-desc {
      margin: 0;
      font-size: 1rem;
      color: var(--theme-meta-color);
    }
  }
}

// 空状态 / 无数据 / 加载 / 错误
.empty-config,
.no-data,
.loading-status,
.error-status,
.no-results {
  padding: 80px 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  @extend .cardHover;
}

.empty-icon {
  font-size: 3rem;
  line-height: 1;
}

.loading-spinner {
  width: 36px;
  height: 36px;
  border: 3px solid var(--flec-border);
  border-top-color: var(--theme-color);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.error-icon {
  font-size: 2.5rem;
}

.retry-btn {
  margin-top: 8px;
  padding: 8px 24px;
  border: 1px solid var(--flec-border);
  border-radius: 6px;
  background: var(--flec-card-bg);
  color: var(--font-color);
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.3s;

  &:hover {
    border-color: var(--theme-color);
    color: var(--theme-color);
  }
}

// 统计面板
.stats-panel {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
  margin-bottom: 24px;

  .stat-card {
    @extend .cardHover;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 4px;
    padding: 20px 12px;
    cursor: default;
    transition: all 0.3s;

    &:last-child {
      cursor: pointer;

      &:hover {
        border-color: var(--theme-color);
        box-shadow: 0 3px 12px 6px rgba(73, 177, 245, 0.1);
      }
    }

    .stat-value {
      font-size: 1.6rem;
      font-weight: bold;
      line-height: 1.2;
    }

    .stat-label {
      font-size: 0.85rem;
      color: var(--theme-meta-color);
    }
  }
}

// 随机文章
.random-article {
  @extend .cardHover;
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 20px 24px;
  margin-bottom: 24px;

  .random-meta {
    display: flex;
    align-items: center;
    gap: 8px;

    .random-avatar {
      width: 28px;
      height: 28px;
      border-radius: 50%;
      object-fit: cover;
    }

    .random-author {
      font-size: 0.9rem;
      font-weight: 600;
    }

    .random-date {
      font-size: 0.8rem;
      color: var(--theme-meta-color);
      margin-left: auto;
    }
  }

  .random-title {
    font-size: 1.15rem;
    font-weight: 600;
    line-height: 1.5;
    text-decoration: none;
    color: var(--font-color);
    transition: color 0.3s;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;

    &:hover {
      color: var(--theme-color);
    }
  }

  .random-btn {
    align-self: flex-start;
    padding: 6px 16px;
    border: 1px solid var(--flec-border);
    border-radius: 6px;
    background: transparent;
    color: var(--theme-meta-color);
    cursor: pointer;
    font-size: 0.85rem;
    transition: all 0.3s;

    &:hover {
      border-color: var(--theme-color);
      color: var(--theme-color);
    }
  }
}

// 筛选栏
.filter-bar {
  @extend .cardHover;
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 16px 20px;
  margin-bottom: 20px;

  .search-wrap {
    position: relative;
    display: flex;
    align-items: center;

    i {
      position: absolute;
      left: 12px;
      font-size: 1rem;
      color: var(--theme-meta-color);
    }

    .search-input {
      width: 100%;
      padding: 8px 12px 8px 36px;
      border: 1px solid var(--flec-border);
      border-radius: 6px;
      background: transparent;
      color: var(--font-color);
      font-size: 0.9rem;
      outline: none;
      transition: border-color 0.3s;

      &:focus {
        border-color: var(--theme-color);
      }

      &::placeholder {
        color: var(--theme-meta-color);
      }
    }
  }

  .author-chips {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;

    .chip {
      padding: 4px 12px;
      border: 1px solid var(--flec-border);
      border-radius: 999px;
      background: transparent;
      color: var(--theme-meta-color);
      cursor: pointer;
      font-size: 0.82rem;
      transition: all 0.3s;
      white-space: nowrap;

      &:hover {
        border-color: var(--theme-color);
        color: var(--theme-color);
      }

      &.active {
        border-color: var(--theme-color);
        background: var(--theme-color);
        color: #fff;
      }

      &.chip-more {
        border-style: dashed;
        font-size: 0.78rem;
        color: var(--theme-meta-color);
        opacity: 0.7;

        &:hover {
          opacity: 1;
          border-color: var(--theme-color);
          color: var(--theme-color);
        }
      }
    }
  }
}

// 文章网格
.article-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 16px;

  .article-card {
    @extend .cardHover;
    display: flex;
    flex-direction: column;
    padding: 20px;
    text-decoration: none;
    color: inherit;
    cursor: pointer;
    transition: all 0.3s;

    &:hover {
      border-color: var(--theme-color);
      transform: translateY(-2px);
      box-shadow: 0 6px 16px 6px rgba(73, 177, 245, 0.08);

      .card-title {
        color: var(--theme-color);
      }
    }

    .card-body {
      display: flex;
      flex-direction: column;
      gap: 12px;
      flex: 1;

      .card-title {
        margin: 0;
        font-size: 1rem;
        font-weight: 600;
        line-height: 1.6;
        transition: color 0.3s;
        display: -webkit-box;
        -webkit-line-clamp: 3;
        -webkit-box-orient: vertical;
        overflow: hidden;
      }

      .card-meta {
        display: flex;
        align-items: center;
        justify-content: space-between;
        gap: 8px;
        margin-top: auto;

        .card-author {
          display: flex;
          align-items: center;
          gap: 6px;
          min-width: 0;

          .author-avatar {
            width: 22px;
            height: 22px;
            border-radius: 50%;
            object-fit: cover;
            flex-shrink: 0;
          }

          .author-name {
            font-size: 0.82rem;
            color: var(--theme-meta-color);
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
          }
        }

        .card-date {
          font-size: 0.78rem;
          color: var(--theme-meta-color);
          flex-shrink: 0;
        }
      }
    }
  }
}

// 响应式
@media screen and (max-width: 1024px) {
  .fish-header {
    padding: 30px;

    .header-content .page-title {
      font-size: 1.75rem;
    }
  }

  .stats-panel {
    grid-template-columns: repeat(2, 1fr);
    gap: 12px;
  }

  .article-grid {
    grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
    gap: 12px;
  }
}

@media screen and (max-width: 768px) {
  .fish-header {
    padding: 20px;

    .header-content .page-title {
      font-size: 1.4rem;
    }

    .header-content .page-desc {
      font-size: 0.9rem;
    }
  }

  .stats-panel {
    .stat-card {
      padding: 14px 8px;

      .stat-value {
        font-size: 1.3rem;
      }
    }
  }

  .random-article {
    padding: 16px 18px;
  }

  .article-grid {
    grid-template-columns: 1fr;
  }

  .filter-bar .author-chips .chip {
    font-size: 0.78rem;
    padding: 3px 10px;
  }
}
</style>
