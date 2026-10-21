<template>
  <transition name="shelf-slide">
    <div
      v-show="!isHidden && showShelf"
      id="shelf"
      :class="{ 'add-padding': addPadding, 'shelf--listing': isListingView }"
      :style="moveWithSidebar"
    >
      <div v-if="showBreadcrumbs" class="shelf-title-row">
        <!-- Selection action bar replaces the title row while items are selected (Drive behaviour). -->
        <div v-if="selectedCount > 0" class="selection-bar" role="toolbar" :aria-label="selectionLabel">
          <button type="button" class="shelf-icon-btn" :aria-label="$t('general.clearSelection')"
            :title="$t('general.clearSelection')" @click="clearSelection">
            <i class="material-symbols">close</i>
          </button>
          <span class="selection-bar__count">{{ selectionLabel }}</span>
          <button v-if="permissions.download" type="button" class="shelf-icon-btn" :aria-label="$t('general.download')"
            :title="$t('general.download')" @click="downloadSelected">
            <i class="material-symbols">download</i>
          </button>
          <button v-if="permissions.modify" type="button" class="shelf-icon-btn" :aria-label="$t('general.move')"
            :title="$t('general.move')" @click="moveSelected">
            <i class="material-symbols">drive_file_move</i>
          </button>
          <button v-if="permissions.delete" type="button" class="shelf-icon-btn" :aria-label="$t('general.delete')"
            :title="$t('general.delete')" @click="deleteSelected">
            <i class="material-symbols">delete</i>
          </button>
          <button type="button" class="shelf-icon-btn" ref="moreButton" :aria-label="$t('buttons.showMore')"
            :title="$t('buttons.showMore')" @click="openSelectionMenu">
            <i class="material-symbols">more_vert</i>
          </button>
        </div>
        <template v-else>
          <breadcrumbs :base="isShare ? `/share/${shareHash}` : undefined" />
          <div v-if="isListingView && !viewModeLocked" class="view-switch" role="group" :aria-label="$t('buttons.switchView')">
            <button v-for="option in viewOptions" :key="option.mode" type="button"
              :class="{ active: activeViewFamily === option.mode }" :aria-pressed="activeViewFamily === option.mode"
              :aria-label="option.label" :title="option.label" @click="setViewMode(option.mode)">
              <i v-if="activeViewFamily === option.mode" class="material-symbols view-switch__check">check</i>
              <i class="material-symbols">{{ option.icon }}</i>
            </button>
          </div>
        </template>
      </div>
      <listing-header v-if="showListingHeader" :hasDuration="hasDuration" />
      <duplicate-finder-actions
        v-if="showDuplicateFinderActions"
        :selectedCount="duplicateFinderSelectedCount"
        :deleting="duplicateFinderDeleting"
        @delete="handleDuplicateFinderDelete"
        @clear="handleDuplicateFinderClear"
      />
    </div>
  </transition>
</template>

<script>
import Breadcrumbs from "@/components/files/Breadcrumbs.vue";
import ListingHeader from "@/components/files/ListingHeader.vue";
import DuplicateFinderActions from "@/components/tools/DuplicateFinderActions.vue";
import { getters, mutations, state } from "@/store";
import { eventBus } from "@/store/eventBus";
import downloadFiles from "@/utils/download";

export default {
  name: "Shelf",
  components: {
    Breadcrumbs,
    ListingHeader,
    DuplicateFinderActions,
  },
  data() {
    return {
      isHidden: false,
      lastScrollTop: 0,
      scrollElement: null,
      scrollFrame: null,
      // Duplicate finder state
      duplicateFinderSelectedCount: 0,
      duplicateFinderDeleting: false,
    };
  },
  computed: {
    isListingView() {
      return getters.shellView() === "listingView";
    },
    selectedItems() {
      if (!this.isListingView || state.selected.length === 0) return [];
      if (typeof state.selected[0] !== "number") return [];
      return state.selected.map((index) => state.req.items?.at(index)).filter((item) => item != null);
    },
    selectedCount() {
      return this.selectedItems.length;
    },
    selectionLabel() {
      const word = this.selectedCount === 1 ? this.$t("general.selectionSingle") : this.$t("general.selectionMultiple");
      return `${this.selectedCount} ${word}`;
    },
    permissions() {
      return { ...getters.globalPermissions(), ...getters.sourcePermissions() };
    },
    viewModeLocked() {
      return getters.viewModeChangeLocked();
    },
    viewOptions() {
      return [
        { mode: "list", icon: "view_list", label: this.$t("buttons.listView") },
        { mode: "normal", icon: "grid_view", label: this.$t("buttons.normalView") },
        { mode: "gallery", icon: "photo_library", label: this.$t("buttons.galleryView") },
      ];
    },
    activeViewFamily() {
      const mode = getters.viewMode();
      if (mode === "list" || mode === "compact") return "list";
      if (mode === "gallery" || mode === "icons") return "gallery";
      return "normal";
    },
    addPadding() {
      return getters.isStickySidebar() || getters.isShare();
    },
    showShelf() {
      if (state.loading.length > 0) {
        return false;
      }
      return this.showBreadcrumbs || this.showListingHeader || this.showDuplicateFinderActions;
    },
    showBreadcrumbs() {
      return getters.showBreadCrumbs();
    },
    isShare() {
      return getters.isShare();
    },
    shareHash() {
      return getters.shareHash();
    },
    showListingHeader() {
      const path = state.route.path || "";
      if (path === "/tools/advancedSearch") {
        return false;
      }
      const itemsLen = state.req?.items?.length ?? 0;
      if (itemsLen <= 0) {
        return false;
      }
      return getters.currentView() === "listingView";
    },
    hasDuration() {
      // Check if any file has duration metadata
      if (!state.req?.items) return false;
      return state.req.items.some(item =>
        item.type !== 'directory' && item.metadata?.duration
      );
    },
    showDuplicateFinderActions() {
      // Show duplicate finder actions when on that route and there are selected items
      return this.duplicateFinderSelectedCount > 0;
    },
    moveWithSidebar() {
      // Match #main.content-card (Layout.vue): 16px shell gutter when the sidebar isn't docked.
      if (this.isListingView && !getters.isMobile() && !(getters.isStickySidebar() && getters.isSidebarVisible())) {
        return { left: "16px" };
      }
      if (getters.isStickySidebar() && getters.isSidebarVisible()) {
        return {
          left: `${state.sidebar.width}em`,
        };
      }
      return {};
    },
  },
  async mounted() {
    eventBus.on('duplicateFinderSelectionChanged', this.handleDuplicateFinderSelectionChanged);
    eventBus.on('duplicateFinderDeletingChanged', this.handleDuplicateFinderDeletingChanged);
    await this.$nextTick();
    this.attachScrollListener();
  },
  beforeUnmount() {
    this.detachScrollListener();

    // Clean up event bus listeners
    eventBus.off('duplicateFinderSelectionChanged', this.handleDuplicateFinderSelectionChanged);
    eventBus.off('duplicateFinderDeletingChanged', this.handleDuplicateFinderDeletingChanged);
  },
  watch: {
    async $route() {
      this.isHidden = false;
      this.lastScrollTop = 0;
      await this.$nextTick();
      this.detachScrollListener();
      this.attachScrollListener();
    },
    showShelf(newValue, oldValue) {
      // When shelf content appears (false -> true), always show it regardless of scroll position
      if (newValue === true && oldValue === false) {
        this.isHidden = false;
      }
    },
  },
  methods: {
    setViewMode(mode) {
      mutations.closeHovers();
      mutations.updateDisplayPreferences({ viewMode: mode });
      void mutations.updateCurrentUser({ viewMode: mode });
    },
    clearSelection() {
      mutations.resetSelected();
    },
    downloadSelected() {
      downloadFiles(this.selectedItems);
    },
    moveSelected() {
      mutations.showPrompt({ name: "move", props: { items: this.selectedItems, operation: "move" } });
    },
    deleteSelected() {
      mutations.showPrompt({ name: "delete", props: { items: this.selectedItems } });
    },
    openSelectionMenu() {
      const rect = this.$refs.moreButton?.getBoundingClientRect();
      mutations.showPrompt({
        name: "ContextMenu",
        props: rect && !getters.isMobile() ? { posX: rect.left, posY: rect.bottom + 4 } : { showCentered: true },
      });
    },
    attachScrollListener() {
      // Try to find the scroll wrapper (used by Scrollbar component)
      const scrollWrapper = document.querySelector('.scroll-wrapper');
      if (scrollWrapper) {
        scrollWrapper.addEventListener('scroll', this.handleScroll, { passive: true });
        this.scrollElement = scrollWrapper;
      }
    },

    detachScrollListener() {
      if (this.scrollElement) {
        this.scrollElement.removeEventListener('scroll', this.handleScroll);
        this.scrollElement = null;
      }
      // Cancel any pending animation frame
      if (this.scrollFrame) {
        cancelAnimationFrame(this.scrollFrame);
        this.scrollFrame = null;
      }
    },

    handleScroll(event) {
      // Use requestAnimationFrame to throttle updates (like Scrollbar component)
      if (this.scrollFrame) return;

      this.scrollFrame = requestAnimationFrame(() => {
        const scrollTop = event.target.scrollTop;

        // Always show when at the top
        if (scrollTop <= 10) {
          this.isHidden = false;
          this.lastScrollTop = scrollTop;
          this.scrollFrame = null;
          return;
        }

        // Calculate scroll difference
        const scrollDiff = scrollTop - this.lastScrollTop;

        // Only trigger if scrolled enough (at least 30px)
        if (Math.abs(scrollDiff) < 30) {
          this.scrollFrame = null;
          return;
        }

        // Hide when scrolling down, show when scrolling up
        if (scrollDiff > 0) {
          // Scrolling down
          this.isHidden = true;
        } else {
          // Scrolling up
          this.isHidden = false;
        }

        this.lastScrollTop = scrollTop;
        this.scrollFrame = null;
      });
    },

    // Duplicate finder event handlers
    handleDuplicateFinderSelectionChanged(count) {
      this.duplicateFinderSelectedCount = count;
    },
    handleDuplicateFinderDeletingChanged(deleting) {
      this.duplicateFinderDeleting = deleting;
    },
    handleDuplicateFinderDelete() {
      eventBus.emit('duplicateFinderDeleteRequested');
    },
    handleDuplicateFinderClear() {
      eventBus.emit('duplicateFinderClearRequested');
    },
  },
};
</script>

<style scoped>
#shelf {
  position: fixed;
  z-index: 1000;
  top: var(--header-height);
  right: 0;
  left: 0;
  box-sizing: border-box;
  height: auto;
  min-height: 0;
  padding: 0 16px;
  overflow: hidden visible;
  background: var(--surfacePrimary);
  pointer-events: auto;
  transition: left 0.2s ease;
}

#shelf.shelf--listing {
  right: 16px;
  border-radius: var(--radius-l) var(--radius-l) 0 0;
}

.shelf-title-row {
  display: flex;
  align-items: center;
  gap: 8px;
  height: 64px;
  min-width: 0;
}

.shelf-title-row > :deep(#breadcrumbs) {
  flex: 1 1 auto;
  min-width: 0;
}

.view-switch {
  display: inline-flex;
  flex: 0 0 auto;
  height: 32px;
  border: 1px solid var(--textSecondary);
  border-radius: var(--radius-full);
  overflow: hidden;
}

.view-switch button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 2px;
  min-width: 44px;
  min-height: 0;
  height: 100%;
  padding: 0 12px;
  border-radius: 0;
  background: transparent;
  color: var(--textPrimary);
  cursor: pointer;
}

.view-switch button + button {
  border-left: 1px solid var(--textSecondary);
}

.view-switch button:hover {
  background: var(--state-hover);
}

.view-switch button.active {
  background: var(--accent-container);
  color: var(--on-accent-container);
}

.view-switch .material-symbols {
  font-size: 18px;
}

.selection-bar {
  display: flex;
  align-items: center;
  gap: 4px;
  width: 100%;
  height: 40px;
  padding: 0 8px 0 4px;
  border-radius: var(--radius-full);
  background: var(--alt-background);
}

.dark-mode .selection-bar {
  background: var(--surfaceSecondary);
}

.selection-bar__count {
  margin: 0 12px 0 4px;
  font-size: 14px;
  font-weight: 500;
  white-space: nowrap;
}

.shelf-icon-btn {
  width: 32px;
  height: 32px;
  min-width: 32px;
  min-height: 32px;
  padding: 0;
  border-radius: var(--radius-full);
  background: transparent;
  color: var(--textPrimary);
  cursor: pointer;
}

.shelf-icon-btn:hover {
  background: var(--state-hover);
}

.shelf-icon-btn .material-symbols {
  font-size: 20px;
}

.shelf-slide-enter-active,
.shelf-slide-leave-active {
  transition: transform 0.3s ease, opacity 0.3s ease !important;
}

.shelf-slide-enter-from,
.shelf-slide-leave-to {
  transform: translateY(-100%);
  opacity: 0;
}

.shelf-slide-enter-to,
.shelf-slide-leave-from {
  transform: translateY(0);
  opacity: 1;
}

@media (width <= 768px) {
  #shelf,
  #shelf.shelf--listing {
    right: 0;
    border-radius: 0;
    padding: 0 8px;
  }

  .view-switch button {
    min-width: 36px;
    padding: 0 8px;
  }

  .view-switch__check {
    display: none;
  }
}
</style>
