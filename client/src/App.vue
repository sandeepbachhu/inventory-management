<template>
  <div class="app">
    <!-- Dim layer behind the open drawer. Rendered only while the drawer is open; CSS shows it on narrow screens only. -->
    <div
      v-if="sidebarOpen"
      class="sidebar-overlay"
      @click="closeSidebar"
    ></div>

    <aside
      id="app-sidebar"
      class="sidebar"
      :class="{ open: sidebarOpen }"
    >
      <div class="sidebar-brand">
        <h1>{{ t('nav.companyName') }}</h1>
        <span class="sidebar-subtitle">{{ t('nav.subtitle') }}</span>
      </div>

      <nav class="sidebar-nav">
        <router-link to="/" class="sidebar-link" :class="{ active: $route.path === '/' }">
          <svg class="sidebar-icon" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <rect x="3" y="3" width="7" height="7" rx="1"/>
            <rect x="14" y="3" width="7" height="7" rx="1"/>
            <rect x="14" y="14" width="7" height="7" rx="1"/>
            <rect x="3" y="14" width="7" height="7" rx="1"/>
          </svg>
          {{ t('nav.overview') }}
        </router-link>
        <router-link to="/inventory" class="sidebar-link" :class="{ active: $route.path === '/inventory' }">
          <svg class="sidebar-icon" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <path d="M21 8l-9-5-9 5v8l9 5 9-5V8z"/>
            <path d="M3 8l9 5 9-5M12 13v8"/>
          </svg>
          {{ t('nav.inventory') }}
        </router-link>
        <router-link to="/orders" class="sidebar-link" :class="{ active: $route.path === '/orders' }">
          <svg class="sidebar-icon" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <path d="M9 5H7a2 2 0 0 0-2 2v12a2 2 0 0 0 2 2h10a2 2 0 0 0 2-2V7a2 2 0 0 0-2-2h-2"/>
            <rect x="9" y="3" width="6" height="4" rx="1"/>
          </svg>
          {{ t('nav.orders') }}
        </router-link>
        <router-link to="/spending" class="sidebar-link" :class="{ active: $route.path === '/spending' }">
          <svg class="sidebar-icon" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <path d="M3 3v18h18"/>
            <path d="M7 15l4-4 3 3 5-6"/>
          </svg>
          {{ t('nav.finance') }}
        </router-link>
        <router-link to="/demand" class="sidebar-link" :class="{ active: $route.path === '/demand' }">
          <svg class="sidebar-icon" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <path d="M3 17l6-6 4 4 8-8"/>
            <path d="M14 7h7v7"/>
          </svg>
          {{ t('nav.demandForecast') }}
        </router-link>
        <router-link to="/reports" class="sidebar-link" :class="{ active: $route.path === '/reports' }">
          <svg class="sidebar-icon" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
            <path d="M14 2v6h6M8 13h8M8 17h8"/>
          </svg>
          {{ t('nav.reports') }}
        </router-link>
      </nav>

      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>

    <div class="content-area">
      <!-- Slim top bar: only the drawer toggle. Hidden on desktop, where the sidebar is always visible. -->
      <header class="topbar">
        <button
          type="button"
          class="menu-toggle"
          :aria-expanded="sidebarOpen ? 'true' : 'false'"
          aria-controls="app-sidebar"
          :aria-label="sidebarOpen ? 'Close navigation' : 'Open navigation'"
          @click="toggleSidebar"
        >
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" aria-hidden="true">
            <path d="M4 6h16M4 12h16M4 18h16"/>
          </svg>
        </button>
      </header>

      <FilterBar />

      <main class="main-content">
        <router-view />
      </main>
    </div>

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
import { ref, onMounted, onBeforeUnmount, computed, watch } from 'vue'
import { useRoute } from 'vue-router'
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
    const route = useRoute()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Mobile drawer state. Below --breakpoint-nav (960px) the 248px sidebar would take too much of a
    // tablet or phone screen, so it becomes an off-canvas drawer that the top bar toggles.
    // sidebarOpen has no visual effect on desktop, where the sidebar is always shown.
    const sidebarOpen = ref(false)
    const closeSidebar = () => {
      sidebarOpen.value = false
    }
    const toggleSidebar = () => {
      sidebarOpen.value = !sidebarOpen.value
    }

    // Close the drawer after navigating, so the newly opened page is visible
    watch(() => route.path, closeSidebar)

    // Escape closes the drawer (keyboard users cannot click the overlay)
    const onKeydown = (event) => {
      if (event.key === 'Escape') closeSidebar()
    }

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
    onMounted(() => window.addEventListener('keydown', onKeydown))
    onBeforeUnmount(() => window.removeEventListener('keydown', onKeydown))

    return {
      t,
      showProfileDetails,
      showTasks,
      tasks,
      sidebarOpen,
      closeSidebar,
      toggleSidebar,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
/*
 * App shell: sidebar on the left, content column on the right.
 * Shared primitives (cards, tables, badges, page header) live in styles/base.css.
 */
.app {
  display: flex;
  align-items: stretch;
  min-height: 100vh;
}

/* ---- Sidebar (desktop: sticky full-height rail) ---- */
.sidebar {
  width: var(--sidebar-w);
  flex-shrink: 0;
  height: 100vh;
  position: sticky;
  top: 0;
  z-index: var(--z-sidebar);
  display: flex;
  flex-direction: column;
  gap: var(--space-4);
  padding: var(--space-5) var(--space-4);
  background: var(--color-surface);
  border-right: 1px solid var(--color-border);
}

.sidebar-brand {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  padding: var(--space-1) var(--space-2) var(--space-4);
  border-bottom: 1px solid var(--color-border);
}

/* No fixed height: long names and Japanese text wrap onto as many lines as they need */
.sidebar-brand h1 {
  font-size: 1.125rem;
  font-weight: 700;
  color: var(--color-ink);
  letter-spacing: -0.025em;
  line-height: 1.35;
  overflow-wrap: break-word;
}

.sidebar-subtitle {
  font-size: 0.813rem;
  color: var(--color-muted);
  line-height: 1.4;
}

/* Only the link list scrolls on very short viewports. The footer sits outside this box,
   so the profile and language menus that open upward are never clipped. */
.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  flex: 1 1 auto;
  min-height: 0;
  overflow-y: auto;
}

.sidebar-link {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  border-radius: var(--radius-md);
  color: var(--color-muted);
  text-decoration: none;
  font-size: 0.938rem;
  font-weight: 500;
  transition: background-color 0.15s ease, color 0.15s ease;
}

.sidebar-icon {
  flex-shrink: 0;
}

.sidebar-link:hover {
  color: var(--color-ink);
  background: var(--color-hover);
}

.sidebar-link.active {
  color: var(--color-accent);
  background: var(--color-accent-soft);
}

.sidebar-footer {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  padding-top: var(--space-4);
  border-top: 1px solid var(--color-border);
}

.sidebar-link:focus-visible,
.menu-toggle:focus-visible {
  outline: 2px solid var(--color-accent);
  outline-offset: 2px;
}

/* ---- Content column ---- */
.content-area {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
}

/* Hidden on desktop. Shown below the nav breakpoint (see media query at the end). */
.topbar {
  display: none;
}

.menu-toggle {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  padding: 0;
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  color: var(--color-text-2);
  cursor: pointer;
  transition: background-color 0.15s ease, border-color 0.15s ease;
}

.menu-toggle:hover {
  background: var(--color-canvas);
  border-color: #cbd5e1;
}

/* Width is capped by the content column, and the content is centered at --content-max */
.main-content {
  flex: 1;
  width: 100%;
  min-width: 0;
  max-width: var(--content-max);
  margin: 0 auto;
  padding: var(--space-6) var(--space-8);
}

.sidebar-overlay {
  display: none;
}

/*
 * Narrow screens (below --breakpoint-nav, 960px): the sidebar becomes an off-canvas drawer.
 * CSS custom properties cannot be used inside @media conditions, so the 960px value is written here.
 * Keep it in sync with --breakpoint-nav in styles/tokens.css. The 959.98px form makes the
 * query match exactly the widths below 960px.
 */
@media (max-width: 959.98px) {
  .sidebar {
    position: fixed;
    top: 0;
    bottom: 0;
    left: 0;
    height: auto;
    max-width: 85vw;
    /* Moved fully out of view when closed. visibility keeps the hidden links out of the tab order. */
    transform: translateX(-100%);
    visibility: hidden;
    transition: transform 0.2s ease, visibility 0.2s ease;
  }

  .sidebar.open {
    transform: none;
    visibility: visible;
    box-shadow: var(--shadow-md);
  }

  /* Sits just below the drawer (z-sidebar - 1) and above the page content */
  .sidebar-overlay {
    display: block;
    position: fixed;
    inset: 0;
    z-index: calc(var(--z-sidebar) - 1);
    background: rgba(15, 23, 42, 0.45);
  }

  .topbar {
    display: flex;
    align-items: center;
    position: sticky;
    top: 0;
    z-index: var(--z-filterbar);
    height: var(--topbar-h);
    padding: 0 var(--space-4);
    background: var(--color-surface);
    border-bottom: 1px solid var(--color-border);
  }

  /* Narrower gutter on phones (at least 16px each side) */
  .main-content {
    padding: var(--space-4);
  }
}
</style>
