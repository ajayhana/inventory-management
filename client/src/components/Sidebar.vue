<template>
  <aside class="sidebar">
    <div class="sidebar-brand">
      <div class="brand-icon">
        <svg width="24" height="24" viewBox="0 0 24 24" fill="none">
          <rect x="2" y="2" width="9" height="9" rx="2" fill="currentColor"/>
          <rect x="13" y="2" width="9" height="9" rx="2" fill="currentColor" opacity="0.6"/>
          <rect x="2" y="13" width="9" height="9" rx="2" fill="currentColor" opacity="0.6"/>
          <rect x="13" y="13" width="9" height="9" rx="2" fill="currentColor" opacity="0.3"/>
        </svg>
      </div>
      <div class="brand-text">
        <span class="brand-name">{{ appTitle }}</span>
        <span v-if="appSubtitle" class="brand-subtitle">{{ appSubtitle }}</span>
      </div>
    </div>

    <nav class="sidebar-nav">
      <router-link
        v-for="route in routes"
        :key="route.path"
        :to="route.path"
        :class="['nav-item', { active: isActive(route.path) }]"
      >
        <span class="nav-icon" v-html="getIcon(route.path)"></span>
        <span class="nav-label">{{ route.label }}</span>
      </router-link>
    </nav>
  </aside>
</template>

<script>
import { useRoute } from 'vue-router'

export default {
  name: 'Sidebar',
  props: {
    routes: { type: Array, required: true },
    appTitle: { type: String, default: 'App' },
    appSubtitle: { type: String, default: '' }
  },
  setup() {
    const route = useRoute()

    const isActive = (path) => {
      if (path === '/') return route.path === '/'
      return route.path.startsWith(path)
    }

    const ICONS = {
      default:   `<svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor"><circle cx="8" cy="8" r="3"/></svg>`,
      overview:  `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><rect x="1" y="1" width="6" height="6" rx="1"/><rect x="9" y="1" width="6" height="6" rx="1"/><rect x="1" y="9" width="6" height="6" rx="1"/><rect x="9" y="9" width="6" height="6" rx="1"/></svg>`,
      dashboard: `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><rect x="1" y="1" width="6" height="6" rx="1"/><rect x="9" y="1" width="6" height="6" rx="1"/><rect x="1" y="9" width="6" height="6" rx="1"/><rect x="9" y="9" width="6" height="6" rx="1"/></svg>`,
      inventory: `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><path d="M8 1L14 4.5V11.5L8 15L2 11.5V4.5L8 1Z"/><path d="M8 8L14 4.5M8 8L2 4.5M8 8V15"/></svg>`,
      order:     `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><rect x="2" y="1" width="12" height="14" rx="1"/><path d="M5 5h6M5 8h6M5 11h4"/></svg>`,
      demand:    `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><polyline points="1,12 5,7 8,9 12,4 15,6"/><polyline points="11,4 15,4 15,8"/></svg>`,
      forecast:  `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><polyline points="1,12 5,7 8,9 12,4 15,6"/><polyline points="11,4 15,4 15,8"/></svg>`,
      spend:     `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><circle cx="8" cy="8" r="6"/><path d="M8 4v1.5M8 10.5V12M5.5 6.5C5.5 5.67 6.67 5 8 5s2.5.67 2.5 1.5S9.33 8 8 8s-2.5.67-2.5 1.5S6.67 11 8 11s2.5-.67 2.5-1.5"/></svg>`,
      finance:   `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><circle cx="8" cy="8" r="6"/><path d="M8 4v1.5M8 10.5V12M5.5 6.5C5.5 5.67 6.67 5 8 5s2.5.67 2.5 1.5S9.33 8 8 8s-2.5.67-2.5 1.5S6.67 11 8 11s2.5-.67 2.5-1.5"/></svg>`,
      report:    `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><rect x="2" y="1" width="12" height="14" rx="1"/><path d="M5 9h1v3H5zM8 6h1v6H8zM11 7h1v5h-1z"/></svg>`,
      analytic:  `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><rect x="2" y="1" width="12" height="14" rx="1"/><path d="M5 9h1v3H5zM8 6h1v6H8zM11 7h1v5h-1z"/></svg>`,
      restock:   `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><polyline points="1,4 1,1 4,1"/><path d="M1 1 C4 1 7 3 8 6"/><polyline points="15,12 15,15 12,15"/><path d="M15 15 C12 15 9 13 8 10"/></svg>`,
      setting:   `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><circle cx="8" cy="8" r="2.5"/><path d="M8 1v2M8 13v2M1 8h2M13 8h2M3.05 3.05l1.41 1.41M11.54 11.54l1.41 1.41M3.05 12.95l1.41-1.41M11.54 4.46l1.41-1.41"/></svg>`,
      user:      `<svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5"><circle cx="8" cy="5" r="3"/><path d="M2 14c0-3.31 2.69-6 6-6s6 2.69 6 6"/></svg>`,
    }

    const getIcon = (path) => {
      const segment = path.replace(/^\//, '').toLowerCase() || 'overview'
      const key = Object.keys(ICONS).find(k => k !== 'default' && segment.includes(k))
      return ICONS[key] || ICONS.default
    }

    return { isActive, getIcon }
  }
}
</script>

<style scoped>
.sidebar {
  width: var(--sidebar-width, 240px);
  min-height: 100vh;
  background: var(--sidebar-bg, #0f172a);
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
  position: sticky;
  top: 0;
  height: 100vh;
  overflow-y: auto;
}

.sidebar-brand {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 1.25rem 1.25rem 1rem;
  border-bottom: 1px solid rgba(255, 255, 255, 0.07);
  margin-bottom: 0.5rem;
}

.brand-icon {
  width: 36px;
  height: 36px;
  background: var(--sidebar-accent, #6366f1);
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  flex-shrink: 0;
}

.brand-text {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.brand-name {
  font-size: 0.9rem;
  font-weight: 700;
  color: #ffffff;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  line-height: 1.2;
}

.brand-subtitle {
  font-size: 0.7rem;
  color: var(--sidebar-text, #94a3b8);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  margin-top: 0.125rem;
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  padding: 0 0.75rem;
  gap: 0.125rem;
  flex: 1;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.6rem 0.75rem;
  border-radius: 7px;
  font-size: 0.875rem;
  font-weight: 500;
  color: var(--sidebar-text, #94a3b8);
  text-decoration: none;
  transition: background 0.15s ease, color 0.15s ease;
  position: relative;
}

.nav-item:hover {
  background: rgba(255, 255, 255, 0.06);
  color: #e2e8f0;
}

.nav-item.active {
  background: var(--sidebar-active-bg, #1e293b);
  color: var(--sidebar-text-active, #ffffff);
}

.nav-item.active::before {
  content: '';
  position: absolute;
  left: 0;
  top: 20%;
  height: 60%;
  width: 3px;
  background: var(--sidebar-accent, #6366f1);
  border-radius: 0 3px 3px 0;
}

.nav-icon {
  display: flex;
  align-items: center;
  flex-shrink: 0;
  opacity: 0.8;
}

.nav-item.active .nav-icon {
  opacity: 1;
}
</style>
