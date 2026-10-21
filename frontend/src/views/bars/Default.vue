<template>
  <header v-if="!isOnlyOffice" :class="['flexbar', 'app-bar']" :inert="overlayShown">
    <div class="app-bar__start" :style="startStyle">
      <action
        v-if="!disableNavButtons"
        class="nav-toggle-button"
        icon="close_back"
        :label="$t('general.close')"
        :disabled="isDisabledMultiAction"
        @action="multiAction"
      />
      <a v-if="showBrand" class="app-brand" :href="homeHref" @click.prevent="goHome">
        <img v-if="brandIcon" :src="brandIcon" alt="" class="app-brand__logo" />
        <span class="app-brand__name">{{ brandName }}</span>
      </a>
    </div>
    <div class="search-bar-container"
         v-if="showSearch && !isSearchActive" @click="openSearch" >
      <i class="material-symbols">search</i>
      <input
        type="text"
        id="search-bar-input"
        :placeholder="$t('general.search', { suffix: '...' })"
        readonly
      />
    </div>
    <title v-if="!showSearch" class="topTitle">{{ getTopTitle }}</title>
    <div class="app-bar__end">
      <action
        v-if="isAdvancedSearchRoute && !disableNavButtons"
        class="menu-button"
        :icon="viewIcon"
        :label="viewModeActionLabel"
        @action="switchView"
        :disabled="viewModeChangeLocked"
      />
      <action
        v-else-if="!isListingView"
        class="overflow-menu-button"
        :icon="iconName"
        :disabled="noItems || isSidebarFloating"
        @click="toggleOverflow"
      />
      <template v-if="showAccount">
        <button
          v-if="darkModeTogglePossible"
          type="button"
          class="app-bar__icon"
          :aria-label="$t('index.toggleDark')"
          :title="$t('index.toggleDark')"
          @click="toggleDarkMode"
        >
          <i class="material-symbols">{{ themeIcon }}</i>
        </button>
        <button
          v-if="settingsAllowed"
          type="button"
          class="app-bar__icon"
          :aria-label="$t('general.settings')"
          :title="$t('index.settingsHover')"
          @click="openSettings"
        >
          <i class="material-symbols">settings</i>
        </button>
        <div class="account" ref="account">
          <button
            type="button"
            class="account__avatar"
            :style="{ background: avatarColor }"
            :aria-label="user.username"
            :title="user.username"
            :aria-expanded="accountOpen"
            @click="accountOpen = !accountOpen"
          >{{ avatarLetter }}</button>
          <transition name="menu-pop">
            <div v-if="accountOpen" class="account__menu" role="menu">
              <div class="account__header">
                <span class="account__avatar account__avatar--large" :style="{ background: avatarColor }">{{ avatarLetter }}</span>
                <span class="account__name">{{ user.username }}</span>
              </div>
              <button v-if="!isMobile" type="button" role="menuitemcheckbox" :aria-checked="isStickySidebar" @click="toggleSticky">
                <i class="material-symbols">dock_to_right</i>
                <span>{{ $t("index.toggleSticky") }}</span>
                <i v-if="isStickySidebar" class="material-symbols account__check">check</i>
              </button>
              <button type="button" role="menuitemcheckbox" :aria-checked="!!user.singleClick" @click="toggleClick">
                <i class="material-symbols">ads_click</i>
                <span>{{ $t("index.toggleClick") }}</span>
                <i v-if="user.singleClick" class="material-symbols account__check">check</i>
              </button>
              <button v-if="settingsAllowed" type="button" role="menuitem" @click="openSettings">
                <i class="material-symbols">manage_accounts</i>
                <span>{{ $t("general.profileSettings") }}</span>
              </button>
              <hr v-if="canLogout" />
              <button v-if="canLogout" type="button" role="menuitem" aria-label="logout-button" @click="logout">
                <i class="material-symbols">logout</i>
                <span>{{ $t("general.logout") }}</span>
              </button>
            </div>
          </transition>
        </div>
      </template>
    </div>
  </header>
</template>

<script>
import { router } from "@/router";
import { getters, state, mutations } from "@/store";
import Action from "@/components/Action.vue";
import { globalVars } from "@/utils/constants";
import { url } from "@/utils";
import { logout } from "@/utils/auth";

const AVATAR_COLORS = ["#1a73e8", "#188038", "#e8710a", "#9334e6", "#d93025", "#00796b", "#c2185b", "#5f6368"];

export default {
  name: "UnifiedHeader",
  components: {
    Action,
  },
  data() {
    return {
      viewModes: ["list", "normal", "icons"],
      accountOpen: false,
    };
  },
  mounted() {
    document.addEventListener("click", this.onDocumentClick);
    document.addEventListener("keydown", this.onDocumentKey);
  },
  beforeUnmount() {
    document.removeEventListener("click", this.onDocumentClick);
    document.removeEventListener("keydown", this.onDocumentKey);
  },
  computed: {
    user() {
      return state.user || { username: "anonymous" };
    },
    startStyle() {
      if (getters.shellView() !== "listingView" || getters.isMobile() || !getters.isSidebarVisible() || !getters.isStickySidebar()) {
        return {};
      }
      return { minWidth: `calc(${getters.sidebarWidth()}em - 8px)` };
    },
    showBrand() {
      return getters.shellView() === "listingView" && !getters.isShare() && !getters.isMobile();
    },
    brandName() {
      return globalVars.name || "FileBrowser";
    },
    brandIcon() {
      return globalVars.loginIcon || "";
    },
    homeHref() {
      return `${globalVars.baseURL || "/"}files/`;
    },
    showAccount() {
      return getters.isLoggedIn() && !getters.isShare() && this.user.username !== "anonymous";
    },
    settingsAllowed() {
      return !state.user?.disableSettings;
    },
    canLogout() {
      return !globalVars.noAuth;
    },
    isDarkMode() {
      return getters.isDarkMode();
    },
    themeIcon() {
      return this.isDarkMode ? "light_mode" : "dark_mode";
    },
    isMobile() {
      return getters.isMobile();
    },
    isStickySidebar() {
      return getters.isStickySidebar();
    },
    darkModeTogglePossible() {
      const enforced = state.shareInfo?.enforceDarkLightMode;
      return enforced !== "dark" && enforced !== "light";
    },
    avatarLetter() {
      return (this.user.username || "?").charAt(0).toUpperCase();
    },
    avatarColor() {
      const name = this.user.username || "";
      let hash = 0;
      for (const ch of name) {
        hash = (hash * 31 + ch.charCodeAt(0)) >>> 0;
      }
      return AVATAR_COLORS[hash % AVATAR_COLORS.length];
    },
    getTopTitle() {
      if (getters.isSettings()) {
        return this.$t("general.settings");
      }
      if (getters.isShare()) {
        if (state.req?.type === "directory" || state.shareInfo?.shareType === "upload") {
          return state.shareInfo?.title;
        }
        return state.req.name;
      }
      const currentTool = getters.currentTool();
      if (currentTool) {
        return currentTool.name;
      }
      return state.req.name;
    },
    disableNavButtons() {
      const isShare = getters.isShare();
      const regularDisabled = globalVars.disableNavButtons && this.isListingView;
      const shareDisabled = isShare && state.shareInfo?.hideNavButtons && getters.currentView() === "listingView";
      const uploadShare = isShare && state.shareInfo?.shareType === "upload"
      return regularDisabled || shareDisabled || uploadShare;
    },
    isOnlyOffice() {
      return getters.currentView() === "onlyOfficeEditor";
    },
    isListingView() {
      return getters.currentView() === "listingView";
    },
    isAdvancedSearchRoute() {
      return (state.route?.path || "") === "/tools/advancedSearch";
    },
    showHeaderSwitchView() {
      return this.isListingView || this.isAdvancedSearchRoute;
    },
    iconName() {
      return getters.currentPromptName() === "OverflowMenu"
        ? "keyboard_arrow_up"
        : "more_vert";
    },
    viewIcon() {
      const icons = {
        list: "view_list",
        compact: "view_list",
        normal: "view_module",
        gallery: "grid_view",
      };
      return icons[getters.viewMode()] || "grid_view";
    },
    isShare() {
      return getters.isShare();
    },
    isSidebarFloating() {
      return getters.isSidebarVisible() && !getters.isStickySidebar();
    },
    noItems() {
      return !state.contextMenuHasItems && !getters.isPreviewView();
    },
    showSearch() {
      return getters.isLoggedIn() && getters.currentView() === "listingView" && !getters.isShare();
    },
    isSearchActive() {
      return state.isSearchActive;
    },
    overlayShown() {
      return state.isSearchActive || getters.currentPromptName() !== "";
    },
    viewModeChangeLocked() {
      return getters.viewModeChangeLocked();
    },
    viewModeActionLabel() {
      if (this.viewModeChangeLocked) {
        return this.$t("profileSettings.enforcedByAdmin");
      }
      return this.$t("buttons.switchView");
    },
    isDisabledMultiAction() {
      return Boolean(state.shareInfo?.disableSidebar) && getters.multibuttonState() === "menu";
    },
    showSwitchView() {
      return this.showHeaderSwitchView;
    },
    showSidebarToggle() {
      return getters.currentView() === "listingView";
    },
    req() {
      return state.req;
    },
    isSettings() {
      return getters.isSettings();
    },
  },
  methods: {
    onDocumentClick(event) {
      if (this.accountOpen && this.$refs.account && !this.$refs.account.contains(event.target)) {
        this.accountOpen = false;
      }
    },
    onDocumentKey(event) {
      if (event.key === "Escape") {
        this.accountOpen = false;
      }
    },
    goHome() {
      void router.push({ path: "/files/" });
    },
    /** Drive-style hamburger: docked -> hidden -> docked (mirrors the Ctrl+B handler in Sidebar.vue). */
    toggleDockedSidebar() {
      const visible = getters.isSidebarVisible();
      if (!visible) {
        void mutations.updateCurrentUser({ stickySidebar: true });
        mutations.toggleSidebar();
      } else if (getters.isStickySidebar()) {
        void mutations.updateCurrentUser({ stickySidebar: false });
        mutations.closeSidebar();
      } else {
        mutations.toggleSidebar();
      }
    },
    toggleDarkMode() {
      mutations.toggleDarkMode();
    },
    toggleClick() {
      void mutations.updateCurrentUser({ singleClick: !state.user.singleClick });
    },
    toggleSticky() {
      if (!state.showSidebar && state.user.stickySidebar) {
        mutations.toggleSidebar();
      }
      void mutations.updateCurrentUser({ stickySidebar: !state.user.stickySidebar });
    },
    openSettings() {
      this.accountOpen = false;
      mutations.setPreviousHistoryItem({
        name: state.req.name,
        source: state.req.source,
        path: state.req.path,
        isShare: getters.isShare(),
      });
      void router.push({ path: "/settings", hash: "#profile-main" });
      mutations.closeTopPrompt();
    },
    async logout() {
      this.accountOpen = false;
      await logout();
    },
    openSearch() {
      const pill = this.$el.querySelector?.(".search-bar-container");
      const root = document.documentElement.style;
      if (pill && !getters.isMobile()) {
        const rect = pill.getBoundingClientRect();
        root.setProperty("--search-anchor-left", `${rect.left}px`);
        root.setProperty("--search-anchor-width", `${rect.width}px`);
        root.setProperty("--search-anchor-shift", "0");
      } else {
        root.removeProperty("--search-anchor-left");
        root.removeProperty("--search-anchor-width");
        root.removeProperty("--search-anchor-shift");
      }
      if (!state.isSearchActive) {
        mutations.closeHovers();
        mutations.closeSidebar();
        mutations.resetSelected();
        mutations.setSearch(true);
        // this is hear to allow for animation
        setTimeout(() => {
          const resultList = document.getElementById("result-list");
          resultList.classList.add("active");
          document.getElementById("search-input").focus();
        }, 100);
      }
    },
    toggleOverflow() {
      if (getters.currentPromptName() === "OverflowMenu") {
        mutations.closeHovers();
      } else {
        mutations.showPrompt({ name: "OverflowMenu" });
      }
    },
    /** Match StatusBar.adjustViewMode: list vs compact, icons vs gallery from gallery size. */
    resolveViewModeForFamily(baseMode) {
      const size = state.user?.gallerySize ?? 5;
      if (baseMode === "list") {
        return size <= 3 ? "compact" : "list";
      }
      if (baseMode === "icons") {
        return size <= 4 ? "icons" : "gallery";
      }
      return baseMode;
    },
    /** Map concrete viewMode to one of the three switch-view families (list|compact → list, etc.). */
    viewModeCycleIndex(mode) {
      if (mode === "list" || mode === "compact") {
        return 0;
      }
      if (mode === "normal") {
        return 1;
      }
      if (mode === "gallery" || mode === "icons") {
        return 2;
      }
      return 1;
    },
    switchView() {
      mutations.closeHovers();
      const current = getters.viewMode();
      const cycleIndex = this.viewModeCycleIndex(current);
      const nextIndex = (cycleIndex + 1) % this.viewModes.length;
      const baseMode = this.viewModes.at(nextIndex);
      const newViewMode = this.resolveViewModeForFamily(baseMode);
      mutations.updateDisplayPreferences({ viewMode: newViewMode });
      void mutations.updateCurrentUser({ viewMode: newViewMode });
    },
    multiAction() {
      const cv = getters.currentView();

      // Check for unsaved editor changes before navigation
      if (cv === "editor" && state.editor.dirty) {
        this.showSaveBeforeExitPrompt(() => this.performNavigation(cv));
        return;
      }

      this.performNavigation(cv);
    },
    performNavigation(cv) {
      if ((cv === "listingView" || cv === "tools") && !getters.isMobile() && !getters.isShare()) {
        this.toggleDockedSidebar();
      } else if (cv === "listingView" || ( getters.isShare() && getters.multibuttonState() !== "close") || cv === "tools") {
        mutations.toggleSidebar();
      } else if (cv === "settings" && getters.isMobile()) {
        mutations.toggleSidebar();
      } else {
        mutations.closeHovers();
        if (cv === "settings") {
          if (state.previousHistoryItem?.name) {
            const prev = state.previousHistoryItem;
            mutations.setPreviousHistoryItem(null);
            url.goToItem(prev.source, prev.path, prev, false, prev.isShare);
            return;
          }
          if (state.shareInfo?.hash && state.req?.source === state.shareInfo.hash) {
            url.goToItem(state.shareInfo.hash, state.req.path, {}, false, true);
            return;
          }
          // otherwise navigate to files
          void router.push({ path: "/files" });
          return;
        }
        if (getters.isPreviewView()) {
          if (state.previousHistoryItem?.name) {
            url.goToItem(state.previousHistoryItem.source, state.previousHistoryItem.path, state.previousHistoryItem, false, state.previousHistoryItem.isShare);
            return;
          } else {
            // navigate to parent directory of current url
            const parentPath = url.removeLastDir(state.route.path);
            void router.push({ path: parentPath });
          }
          return;
        }

        router.go(-1);
      }
    },
    showSaveBeforeExitPrompt(onConfirmAction) {
      mutations.showPrompt({
        name: "SaveBeforeExit",
        pinned: true,
        confirm: async () => {
          // Save and exit - trigger the save action
          // If save fails, this will throw and be caught by SaveBeforeExit component
          if (!state.editor.saveHandler) {
            throw new Error("No editor save handler registered");
          }
          await state.editor.saveHandler();
          mutations.setEditorDirty(false);
          onConfirmAction();
        },
        discard: () => {
          // Discard changes and exit
          mutations.setEditorDirty(false);
          onConfirmAction();
        },
        cancel: () => {
          // Keep editing - do nothing
        },
      });
    },
  },
};
</script>

<style scoped>
header.app-bar {
  background-color: var(--background);
  color: var(--textPrimary);
  gap: 0.5rem;
  padding-left: calc(8px + var(--safe-area-left));
  padding-right: calc(16px + var(--safe-area-right));
}

.app-bar__start,
.app-bar__end {
  display: flex;
  align-items: center;
  gap: 4px;
  flex: 0 0 auto;
}

.app-bar__start {
  min-width: 0;
}

.app-bar__end {
  justify-content: flex-end;
}

:deep(.action.nav-toggle-button .line),
:deep(.action.menu-button),
:deep(.action.overflow-menu-button) {
  stroke: var(--textSecondary);
  color: var(--textSecondary);
}

:deep(.action) {
  border-radius: var(--radius-full);
}

:deep(.action:hover) {
  background: var(--state-hover);
}

.app-brand {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 0 8px;
  min-width: 0;
  height: 48px;
  color: var(--textSecondary);
  text-decoration: none;
}

.app-brand__logo {
  width: 40px;
  height: 40px;
  object-fit: contain;
}

.app-brand__name {
  font-family: var(--font-display);
  font-size: 22px;
  line-height: 28px;
  font-weight: 400;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.search-bar-container {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1 1 auto;
  max-width: 720px;
  min-width: 0;
  height: 48px;
  margin-right: auto;
  padding: 0 16px 0 12px;
  box-sizing: border-box;
  border-radius: var(--radius-full);
  background-color: var(--search-bg);
  cursor: text;
  transition: background-color 0.15s ease, box-shadow 0.15s ease;
}

.search-bar-container:hover {
  background-color: color-mix(in srgb, var(--search-bg) 88%, var(--textPrimary));
}

.search-bar-container .material-symbols {
  font-size: 24px;
  color: var(--textSecondary);
  user-select: none;
}

#search-bar-input {
  width: 100%;
  border: none;
  outline: none;
  background: transparent;
  color: var(--textPrimary);
  font: inherit;
  font-size: 16px;
  user-select: none;
  cursor: text;
}

#search-bar-input::placeholder {
  color: var(--textSecondary);
}

.app-bar__icon {
  width: 40px;
  height: 40px;
  min-width: 40px;
  padding: 0;
  border-radius: var(--radius-full);
  background: transparent;
  color: var(--textSecondary);
  cursor: pointer;
}

.app-bar__icon:hover {
  background: var(--state-hover);
}

.app-bar__icon .material-symbols {
  font-size: 24px;
}

.account {
  position: relative;
  margin-left: 4px;
}

.account__avatar {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  min-width: 32px;
  min-height: 32px;
  padding: 0;
  border-radius: 50%;
  color: #fff;
  font-family: var(--font-display);
  font-size: 15px;
  font-weight: 500;
  cursor: pointer;
  outline: 4px solid transparent;
  transition: outline-color 0.15s ease;
}

button.account__avatar:hover {
  outline-color: var(--state-hover);
}

.account__avatar--large {
  width: 64px;
  height: 64px;
  font-size: 30px;
  cursor: default;
}

.account__menu {
  position: absolute;
  top: calc(100% + 8px);
  right: 0;
  z-index: 10;
  width: 300px;
  padding: 8px 0;
  border-radius: var(--radius-xl);
  background: var(--alt-background);
  box-shadow: var(--elevation-2);
}

.account__header {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  padding: 16px 16px 12px;
}

.account__name {
  font-family: var(--font-display);
  font-size: 18px;
  font-weight: 400;
  color: var(--textPrimary);
}

.account__menu button {
  display: flex;
  align-items: center;
  justify-content: flex-start;
  gap: 16px;
  width: 100%;
  height: 48px;
  padding: 0 24px;
  border-radius: 0;
  background: none;
  color: var(--textPrimary);
  font: inherit;
  font-size: 14px;
  text-align: left;
  cursor: pointer;
}

.account__menu button:hover {
  background: var(--state-hover);
}

.account__menu button span {
  flex: 1;
}

.account__menu .material-symbols {
  font-size: 20px;
  color: var(--textSecondary);
}

.account__menu .account__check {
  color: var(--accent);
}

.account__menu hr {
  margin: 8px 0;
  border: 0;
  border-top: 1px solid var(--divider);
}

.menu-pop-enter-active,
.menu-pop-leave-active {
  transition: opacity 0.12s ease, transform 0.12s ease;
}

.menu-pop-enter-from,
.menu-pop-leave-to {
  opacity: 0;
  transform: translateY(-4px) scale(0.98);
}

@media (width <= 768px) {
  .search-bar-container {
    height: 44px;
  }
  .app-bar__icon {
    display: none;
  }
}
</style>
