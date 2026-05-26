<template>
  <div class="app">
    <!-- Sidebar -->
    <aside class="sidebar" :class="{ collapsed: sidebarCollapsed }">
      <div class="sidebar-header">
        <div class="brand-name">
          <span class="brand-full">{{ t('nav.companyName') }}</span>
          <span class="brand-initials">CC</span>
        </div>
        <div class="brand-sub">{{ t('nav.subtitle') }}</div>
      </div>

      <nav class="sidebar-nav">
        <router-link
          v-for="item in navItems"
          :key="item.path"
          :to="item.path"
          class="sidebar-link"
          :class="{ active: $route.path === item.path }"
          :title="item.label"
        >
          <svg width="16" height="16" viewBox="0 0 16 16" fill="none"
               stroke="currentColor" stroke-width="1.5"
               stroke-linecap="round" stroke-linejoin="round">
            <path :d="item.icon" />
          </svg>
          <span class="link-label">{{ item.label }}</span>
        </router-link>

        <button class="sidebar-toggle" @click="toggleSidebar" :title="sidebarCollapsed ? 'Expand sidebar' : 'Collapse sidebar'">
          <svg width="16" height="16" viewBox="0 0 16 16" fill="none"
               stroke="currentColor" stroke-width="1.5"
               stroke-linecap="round" stroke-linejoin="round">
            <!-- Double chevron left when expanded, double chevron right when collapsed -->
            <path :d="sidebarCollapsed ? 'M6 3l5 5-5 5M3 3l5 5-5 5' : 'M10 3l-5 5 5 5M13 3l-5 5 5 5'" />
          </svg>
          <span class="link-label">Collapse</span>
        </button>
      </nav>

      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>

    <!-- Main area -->
    <div class="main-wrapper">
      <FilterBar />
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <!-- Modals (unchanged) -->
    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />
    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, onMounted, computed } from 'vue'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()

    // Sidebar navigation items — icon paths are inline SVG path data (16×16 viewBox)
    const navItems = [
      { path: '/',           label: t('nav.overview'),       icon: 'M2 10 L8 4 L14 10 V14 H10 V11 H6 V14 H2 Z' },
      { path: '/inventory',  label: t('nav.inventory'),      icon: 'M2 4h12v2H2zm0 5h12v2H2zm0 5h12v2H2z' },
      { path: '/orders',     label: t('nav.orders'),         icon: 'M3 3h10l1 4H2zm1 4v6h8V7' },
      { path: '/spending',   label: t('nav.finance'),        icon: 'M8 2a6 6 0 100 12A6 6 0 008 2zm0 2v4l3 2' },
      { path: '/demand',     label: t('nav.demandForecast'), icon: 'M2 12 L6 8 L9 10 L14 4' },
      { path: '/reports',    label: 'Reports',               icon: 'M3 3h10v10H3zm2 2v6h2V5zm3 2v4h2V7z' },
      { path: '/restocking', label: 'Restocking',            icon: 'M8 2v5l3-3m-3 3L5 4M2 13h12' },
    ]

    // Sidebar collapse state — persisted in localStorage, default collapsed on narrow screens
    const sidebarCollapsed = ref(
      window.innerWidth <= 900
        ? true
        : localStorage.getItem('sidebar-collapsed') === 'true'
    )

    const toggleSidebar = () => {
      sidebarCollapsed.value = !sidebarCollapsed.value
      localStorage.setItem('sidebar-collapsed', sidebarCollapsed.value)
    }

    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(loadTasks)

    return {
      t,
      navItems,
      sidebarCollapsed,
      toggleSidebar,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

:root {
  /* Sidebar */
  --sidebar-width: 240px;
  --sidebar-bg: #0f172a;
  --sidebar-border: #1e293b;
  --sidebar-link-color: #94a3b8;
  --sidebar-link-hover-bg: #1e293b;
  --sidebar-link-hover-text: #e2e8f0;
  --sidebar-link-active-bg: #1e3a5f;
  --sidebar-link-active-text: #60a5fa;
  /* Spacing tokens */
  --space-xs: 4px;
  --space-sm: 8px;
  --space-md: 16px;
  --space-lg: 24px;
  --space-xl: 32px;
  --space-2xl: 48px;
}

html, body {
  height: 100%;
  overflow: hidden;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  background: #f8fafc;
  color: #1e293b;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#app {
  height: 100vh;
}

/* ── App shell ── */
.app {
  display: flex;
  height: 100vh;
}

/* ── Sidebar ── */
.sidebar {
  width: var(--sidebar-width);
  min-width: var(--sidebar-width);
  height: 100vh;
  background: var(--sidebar-bg);
  border-right: 1px solid var(--sidebar-border);
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  flex-shrink: 0;
}

.sidebar-header {
  padding: 24px 20px 20px;
  border-bottom: 1px solid var(--sidebar-border);
}

.brand-name {
  font-size: 0.9375rem;
  font-weight: 700;
  color: #f1f5f9;
  letter-spacing: -0.01em;
  line-height: 1.3;
}

.brand-sub {
  font-size: 0.75rem;
  color: #64748b;
  margin-top: 3px;
  font-weight: 400;
}

/* ── Sidebar nav ── */
.sidebar-nav {
  flex: 1;
  padding: 16px 12px;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.sidebar-link {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 9px 12px;
  border-radius: 8px;
  color: var(--sidebar-link-color);
  font-size: 0.875rem;
  font-weight: 500;
  text-decoration: none;
  transition: background 0.15s, color 0.15s;
  white-space: nowrap;
}

.sidebar-link:hover {
  background: var(--sidebar-link-hover-bg);
  color: var(--sidebar-link-hover-text);
}

.sidebar-link.active {
  background: var(--sidebar-link-active-bg);
  color: var(--sidebar-link-active-text);
}

.sidebar-link svg {
  flex-shrink: 0;
  opacity: 0.8;
}

.sidebar-link.active svg {
  opacity: 1;
}

/* ── Sidebar footer ── */
.sidebar-footer {
  padding: 12px;
  border-top: 1px solid var(--sidebar-border);
  display: flex;
  flex-direction: column;
  gap: 8px;
}

/* ── Main wrapper ── */
.main-wrapper {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: #f8fafc;
}

/* ── Main content ── */
.main-content {
  flex: 1;
  overflow-y: auto;
  padding: 28px 32px;
}

/* ── Page header ── */
.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.375rem;
  letter-spacing: -0.025em;
}

.page-header p {
  color: #64748b;
  font-size: 0.938rem;
}

/* ── Stats grid ── */
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: white;
  padding: 1.25rem;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
  transition: all 0.2s ease;
}

.stat-card:hover {
  border-color: #cbd5e1;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.06);
}

.stat-label {
  color: #64748b;
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.stat-card.warning .stat-value { color: #ea580c; }
.stat-card.success .stat-value { color: #059669; }
.stat-card.danger  .stat-value { color: #dc2626; }
.stat-card.info    .stat-value { color: #2563eb; }

/* ── Cards ── */
.card {
  background: white;
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid #e2e8f0;
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

/* ── Tables ── */
.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
  border-bottom: 1px solid #e2e8f0;
}

th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  font-weight: 600;
  color: #475569;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: 0.5rem 0.75rem;
  border-top: 1px solid #f1f5f9;
  color: #334155;
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: #f8fafc;
}

/* ── Badges ── */
.badge {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success    { background: #d1fae5; color: #065f46; }
.badge.warning    { background: #fed7aa; color: #92400e; }
.badge.danger     { background: #fecaca; color: #991b1b; }
.badge.info       { background: #dbeafe; color: #1e40af; }
.badge.increasing { background: #d1fae5; color: #065f46; }
.badge.decreasing { background: #fecaca; color: #991b1b; }
.badge.stable     { background: #e0e7ff; color: #3730a3; }
.badge.high       { background: #fecaca; color: #991b1b; }
.badge.medium     { background: #fed7aa; color: #92400e; }
.badge.low        { background: #dbeafe; color: #1e40af; }

/* ── State helpers ── */
.loading {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}

/* ── Sidebar collapsed mode ── */
.sidebar {
  transition: width 0.2s ease, min-width 0.2s ease;
}

.sidebar.collapsed {
  width: 56px;
  min-width: 56px;
}

/* Hide text labels when collapsed */
.sidebar.collapsed .link-label,
.sidebar.collapsed .brand-sub,
.sidebar.collapsed .brand-full {
  display: none;
}

/* Show initials when collapsed */
.sidebar.collapsed .brand-initials {
  display: block;
}

/* Always hide initials when expanded */
.brand-initials {
  display: none;
  font-size: 0.9375rem;
  font-weight: 700;
  color: #f1f5f9;
}

/* Collapsed: center icons */
.sidebar.collapsed .sidebar-link,
.sidebar.collapsed .sidebar-toggle {
  justify-content: center;
  padding: 9px 0;
}

/* Collapsed: center brand */
.sidebar.collapsed .sidebar-header {
  display: flex;
  justify-content: center;
  padding: 20px 0;
}

/* Collapsed: hide footer text (LanguageSwitcher text, profile name) */
.sidebar.collapsed .sidebar-footer {
  align-items: center;
  padding: 12px 4px;
}

/* Toggle button — same style as a nav link */
.sidebar-toggle {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 9px 12px;
  border-radius: 8px;
  color: #64748b;
  font-size: 0.875rem;
  font-weight: 500;
  background: none;
  border: none;
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
  width: 100%;
  margin-top: 8px;
  font-family: inherit;
}

.sidebar-toggle:hover {
  background: #1e293b;
  color: #e2e8f0;
}

/* Tooltip for collapsed nav items */
.sidebar.collapsed .sidebar-link {
  position: relative;
}

.sidebar.collapsed .sidebar-link::after {
  content: attr(title);
  position: absolute;
  left: calc(100% + 12px);
  top: 50%;
  transform: translateY(-50%);
  background: #1e293b;
  color: #e2e8f0;
  font-size: 0.8125rem;
  font-weight: 500;
  padding: 5px 10px;
  border-radius: 6px;
  white-space: nowrap;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.15s;
  z-index: 200;
  border: 1px solid #334155;
}

.sidebar.collapsed .sidebar-link:hover::after {
  opacity: 1;
}
</style>
