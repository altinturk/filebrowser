<template>
  <div class="sidebar-top">
    <!-- Drive-style primary action: opens the create menu anchored under the button. -->
    <transition v-if="shareInfo.shareType !== 'upload'" name="expand" @before-enter="beforeEnter" @enter="enter"
      @leave="leave">
      <div v-if="!hideSidebarFileActions && isListingView" class="new-button-wrap">
        <button type="button" @click="openContextMenu" aria-label="File-Actions" data-testid="file-actions-button"
          class="new-button" ref="newButton" :title="$t('sidebar.fileActions')">
          <i class="material-symbols">add</i>
          <span>{{ $t("general.new", { suffix: "" }) }}</span>
        </button>
      </div>
    </transition>
    <!-- Hidden marker for tests to detect when file actions should be available -->
    <div v-if="isDataLoaded && isListingView && shareInfo.shareType !== 'upload'"
         data-testid="file-actions-ready"
         style="display: none;"
         :data-hidden="hideSidebarFileActions">
    </div>

    <!-- Shares have no app-bar account menu, so keep their quick controls here. -->
    <div v-if="isShare && isDataLoaded && !disableQuickToggles" class="share-controls" @mouseleave="hideTooltip">
      <button v-if="user.username === 'anonymous' && shouldShowLogin" type="button" class="share-control"
        aria-label="Login" @click="navigateToLogin">
        <i class="material-symbols">login</i>
        <span>{{ $t("general.login") }}</span>
      </button>
      <button v-else-if="canLogout" type="button" class="share-control" aria-label="logout-button" @click="logout">
        <i class="material-symbols">logout</i>
        <span>{{ $t("general.logout") }}</span>
      </button>
      <button v-if="darkModeTogglePossible" type="button" class="share-control share-control--icon"
        aria-label="Toggle Theme" :class="{ active: user?.darkMode }" @click="toggleDarkMode"
        @mouseenter="showTooltip($event, $t('index.toggleDark'))">
        <i class="material-symbols">dark_mode</i>
      </button>
    </div>
  </div>

  <SidebarLinks />
</template>

<script>
import { logout } from "@/utils/auth";
import { globalVars } from "@/utils/constants";
import { state, getters, mutations } from "@/store";
import {
  hideInteractiveTooltip,
  showHoverTooltip,
} from "@/utils/tooltipHelp.js";
import SidebarLinks from "./Links.vue";
import { url } from "@/utils";
import { expandLeave } from "@/utils/expandTransition";

export default {
  name: "SidebarGeneral",
  components: {
    SidebarLinks,
  },
  data() {
    return {};
  },
  computed: {
    // Check if data is loaded before showing user info
    isDataLoaded() {
      if (getters.isShare()) {
        // For shares, wait for shareInfo to be loaded
        return state.shareInfo !== null && state.shareInfo !== undefined;
      }
      // For regular files, user should be loaded
      return state.user !== null && state.user !== undefined;
    },
    hasCreateOptions() {
      if (getters.isShare()) {
        return state.shareInfo?.allowCreate === true
      }
      const global = getters.globalPermissions();
      return getters.sourcePermissions().create || global.share || global.admin;
    },
    shareInfo: () => state.shareInfo,
    disableQuickToggles: () => state.user?.disableQuickToggles,
    hideSidebarFileActions() {
      return state.user?.hideSidebarFileActions || getters.isInvalidShare() || !this.hasCreateOptions;
    },
    settingsAllowed: () => !state.user?.disableSettings,
    isSettings: () => getters.isSettings(),
    isStickySidebar: () => getters.isStickySidebar(),
    isMobile: () => getters.isMobile(),
    isInvalidShare: () => getters.isInvalidShare(),
    isListingView: () => getters.shellView() === "listingView",
    user: () => (state.user || {username: 'anonymous'}),
    isShare: () => getters.isShare(),
    active: () => getters.isSidebarVisible(),
    canLogout: () => !globalVars.noAuth && state.user?.username !== 'anonymous',
    route: () => state.route,
    realtimeActive: () => state.realtimeActive,
    darkModeTogglePossible: () => state.shareInfo?.enforceDarkLightMode !== "dark" && state.shareInfo?.enforceDarkLightMode !== "light",
    shouldShowLogin() {
      if (getters.isShare()) {
        // Don't show login until shareInfo is fully loaded
        if (state.shareInfo?.disableLoginOption) {
          return false;
        }
      }
      return true;
    },
  },
  watch: {
    route() {
      if (!getters.isLoggedIn()) {
        return;
      }
      if (!this.isStickySidebar && !state.shareInfo?.singleFileShare) {
        mutations.closeSidebar();
      }
    },
  },
  methods: {
    openContextMenu() {
      mutations.resetSelected();
      const rect = this.$refs.newButton?.getBoundingClientRect();
      mutations.showPrompt({
        name: "ContextMenu",
        props: rect && !this.isMobile
          ? { createOnly: true, posX: rect.left, posY: rect.bottom + 4 }
          : { showCentered: true, createOnly: true },
      });
    },
    checkLogin() {
      return getters.isLoggedIn() && !getters.routePath().startsWith("/share");
    },
    toggleClick() {
      void mutations.updateCurrentUser({ singleClick: !state.user.singleClick });
    },
    toggleDarkMode() {
      mutations.toggleDarkMode();
    },
    toggleSticky() {
      // keep sidebar open if disabling sticky sidebar
      if (!state.showSidebar && state.user.stickySidebar) {
        mutations.toggleSidebar();
      }
      void mutations.updateCurrentUser({ stickySidebar: !state.user.stickySidebar });
    },
    navigateTo(path,hash) {
      mutations.setPreviousHistoryItem({
        name: state.req.name,
        source: state.req.source,
        path: state.req.path,
        isShare: getters.isShare()
      });
      this.$router.push({ path: path, hash: hash });
      mutations.closeTopPrompt();
    },
    navigateToLogin() {
      this.$router.push({ path: "/login", query: { redirect: this.$route.path } });
    },
    // Show the help overlay
    help() {
      mutations.showPrompt("help");
    },

    // Logout the user
    async logout() {
      const isShare = getters.isShare();
      let sharePath;

      if (isShare && state.shareInfo?.hash) {
        const shareBase = `${globalVars.baseURL}public/share/${state.shareInfo.hash}`;
        const subPath = state.req?.path && state.req.path !== '/'
          ? url.removeLeadingSlash(state.req.path)
          : '';
        sharePath = subPath ? `${shareBase}/${url.encodedPath(subPath)}` : shareBase;
      }
      await logout(sharePath);
    },
    beforeEnter(el) {
      el.style.maxHeight = '0';
      el.style.opacity = '0';
    },
    enter(el, done) {
      requestAnimationFrame(() => {
        el.style.transition = 'max-height 0.2s ease, opacity 0.15s ease';
        el.style.maxHeight = `${el.scrollHeight}px`;
        el.style.opacity = '1';
        const onTransitionEnd = () => {
          el.style.maxHeight = '';
          el.removeEventListener('transitionend', onTransitionEnd);
          done();
        };
        el.addEventListener('transitionend', onTransitionEnd);
      });
    },
    leave(el, done) {
      // Finish on a timer, not transitionend: a card shown on first render never ran enter() and has no
      // transition, and sidebar resizing sets `transition: none`. Either way transitionend never fires,
      // done() is never called, and the card stays mounted, invisible but still clickable.
      expandLeave(el, done, 200);
    },
    showTooltip(event, text) {
      if (text) {
        showHoverTooltip(text, event);
      }
    },
    hideTooltip() {
      hideInteractiveTooltip();
    },
  },
};
</script>

<style>
.sidebar-top {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 0 0 8px;
}

.new-button-wrap {
  padding: 4px 0 8px;
}

.new-button {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  height: 44px;
  padding: 0 18px 0 12px;
  border: 0;
  border-radius: var(--radius-m);
  background: var(--surfacePrimary);
  color: var(--textPrimary);
  box-shadow: var(--elevation-1);
  font-family: var(--font-display);
  font-size: 14px;
  font-weight: 500;
  letter-spacing: 0.01em;
  cursor: pointer;
  transition: box-shadow 0.15s ease, background-color 0.15s ease;
}

.dark-mode .new-button {
  background: var(--surfaceSecondary);
}

.new-button:hover {
  box-shadow: var(--elevation-2);
  background: color-mix(in srgb, var(--surfacePrimary) 92%, var(--accent));
}

.dark-mode .new-button:hover {
  background: color-mix(in srgb, var(--surfaceSecondary) 88%, var(--accent));
}

.new-button .material-symbols {
  padding: 0;
  font-size: 22px;
}

.share-controls {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 0 4px;
}

.share-control {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  height: 36px;
  padding: 0 16px;
  border-radius: var(--radius-full);
  background: transparent;
  border: 1px solid var(--divider);
  color: var(--textPrimary);
  font: inherit;
  cursor: pointer;
}

.share-control:hover {
  background: var(--state-hover);
}

.share-control--icon {
  width: 36px;
  padding: 0;
  justify-content: center;
}

.share-control.active {
  background: var(--accent-container);
  color: var(--on-accent-container);
  border-color: transparent;
}

.share-control .material-symbols {
  font-size: 20px;
}
</style>
