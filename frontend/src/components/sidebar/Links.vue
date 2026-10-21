<template>
  <div class="sidebar-links card">
    <!-- Folders: scrollable tree rooted at the current source (Drive's "My Drive" section). -->
    <div class="sidebar-tree-section">
      <div class="nav-section-label">
        <span>{{ $t("general.folders") }}</span>
      </div>
      <!-- Source switcher only when multiple sources (hidden in shares) -->
      <div
        v-if="!isShare && sourceNames.length > 1"
        class="source-card-container"
        ref="sourceCardContainer"
      >
        <ExpandDropdown
          class="sidebar-source-switcher"
          :model-value="activeSource"
          :options="sourceDropdownOptions"
          :aria-label="$t('general.source')"
          @update:model-value="navigateToSource"
        />
      </div>
      <router-link v-if="showTreeRoot" :to="treeRootTo" class="tree-root" :class="{ active: isAtTreeRoot }"
        :title="treeRootLabel">
        <svg v-if="!isShare" class="realtime-pulse" :class="treeRootStatusClass">
          <circle class="center" cx="50%" cy="50%" r="7px"></circle>
          <circle class="pulse" cx="50%" cy="50%" r="10px"></circle>
        </svg>
        <i v-else class="material-symbols tree-root__icon">folder_shared</i>
        <span>{{ treeRootLabel }}</span>
      </router-link>
      <FileTree
        :currentPath="currentPath"
        :currentSource="currentSource"
        class="file-tree-container"
      />
    </div>

    <!-- Links: pinned to the bottom of the sidebar, above the footer. -->
    <div class="sidebar-links-section">
      <div class="nav-section-label">
        <span>{{ $t("general.links") }}</span>
        <button v-if="isShare" type="button" aria-label="Edit Share" class="nav-section-action"
          :disabled="!canEdit"
          @mouseenter="showTooltip($event, editShareText)" @mouseleave="hideTooltip"
          @click="showEditShareHover"><i class="material-symbols">edit</i></button>
        <button v-else type="button" aria-label="Edit Links" class="nav-section-action"
          @mouseenter="showTooltip($event, $t('sidebar.customizeLinks'))" @mouseleave="hideTooltip"
          @click="openSidebarLinksPrompt"><i class="material-symbols">edit</i></button>
      </div>
        <!-- Share Info section - shown when viewing a share -->
        <div v-if="isShare && !disableShareCard" class="share-info-section">
          <ShareInfo :hash="hash" :token="token" :sub-path="subPath" />
        </div>
        <transition-group name="expand" tag="div" class="inner-card">
          <template v-for="(link, index) in orderedSidebarLinks" :key="`link-${index}-${link.category}`">
            <!-- Divider: renders as hr line or text label -->
            <div v-if="link.category === 'divider'" class="sidebar-divider-container">
              <hr v-if="!link.name || link.name.toLowerCase() === 'divider'" class="sidebar-divider" />
              <span v-else class="sidebar-divider-text">{{ link.name }}</span>
            </div>
            <!-- Source Location custom link -->
            <a v-else-if="link.category === 'custom' && link.name === 'sourceLocation'" :href="link.target"
              :aria-label="link.name" class="action button sidebar-link-button" @click.prevent="handleLinkClick(link)">
              <div class="link-container">
                <i class="material-symbols link-icon">open_in_new</i>
                <span>{{ $t('buttons.goToSource') }}</span>
              </div>
            </a>
            <!-- Source-type links (source, source-minimal, source-alt, source-hybrid, source-hybrid-2); usage bar hidden for source-minimal -->
            <a v-else-if="isSourceCategory(link.category)" :href="getLinkHref(link)"
              class="action button source-button sidebar-link-button" :class="{
                active: isLinkActive(link),
                disabled: !isLinkAccessible(link)
              }" @click.prevent="handleLinkClick(link)" :aria-label="link.name">
              <div class="source-container" :class="{ 'has-usage-info': hasUsageInfo(link) }">
                <!-- Show custom icon if user has set one -->
                <i v-if="link.icon" :class="`${getIconClass(link.icon)} link-icon`">{{ link.icon }}</i>
                <!-- Otherwise show animated status indicator -->
                <svg v-else-if="isLinkAccessible(link)" class="realtime-pulse" :class="{
                  active: realtimeActive,
                  danger: sourceInfo[link.sourceName]?.status !== 'indexing' && sourceInfo[link.sourceName]?.status !== 'ready',
                  warning: sourceInfo[link.sourceName]?.status === 'indexing',
                  ready: sourceInfo[link.sourceName]?.status === 'ready',
                }">
                  <circle class="center" cx="50%" cy="50%" r="7px"></circle>
                  <circle class="pulse" cx="50%" cy="50%" r="10px"></circle>
                </svg>
                <HelpTooltipIcon
                  v-else
                  icon="warning"
                  icon-style="symbols"
                  icon-class="warning-icon"
                  :text="$t('sidebar.sourceNotAccessible')"
                />
                <span>{{ link.name }}</span>
                <i v-if="hasUsageInfo(link)" class="no-select material-symbols-outlined tooltip-info-icon"
                  :class="{ 'tooltip-info-icon--pressed': pressedSourceInfo === link.sourceName }"
                  role="button"
                  tabindex="0"
                  @touchstart.stop="onSourceInfoTouchStart(link.sourceName)"
                  @touchend.stop="onSourceInfoTouchEnd($event, sourceInfo[link.sourceName] || {})"
                  @touchcancel.stop="onSourceInfoTouchCancel"
                  @click.stop="onSourceInfoClick($event, sourceInfo[link.sourceName] || {})"
                  @mouseenter="onSourceInfoMouseEnter($event, sourceInfo[link.sourceName] || {})"
                  @mouseleave="onSourceInfoMouseLeave">
                  info
                </i>
              </div>
              <div v-if="hasUsageInfo(link) && baseSidebarCategory(link.category) !== 'source-minimal'" class="usage-info">
                <!-- For source-hybrid, show single bar with background value for disk usage -->
                <ProgressBar
                  v-if="isHybridCategory(link.category)"
                  :key="`progress-hybrid-${link.sourceName}-${sourceInfo[link.sourceName]?.used || 0}-${getDiskUsed(link, sourceInfo[link.sourceName] || {})}-${getUsageTotal(link, sourceInfo[link.sourceName] || {})}`"
                  :val="sourceInfo[link.sourceName]?.used || 0"
                  :val-background="getDiskUsed(link, sourceInfo[link.sourceName] || {})"
                  :val-text="baseSidebarCategory(link.category) === 'source-hybrid-2' ? getDiskUsed(link, sourceInfo[link.sourceName] || {}) : null"
                  :max="getUsageTotal(link, sourceInfo[link.sourceName] || {})"
                  :status="getProgressBarStatus(link, sourceInfo[link.sourceName] || {})"
                  unit="bytes">
                </ProgressBar>
                <!-- For other source types, show single bar -->
                <ProgressBar
                  v-else
                  :key="`progress-${link.sourceName}-${sourceInfo[link.sourceName]?.used || 0}-${getDiskUsed(link, sourceInfo[link.sourceName] || {})}-${getUsageTotal(link, sourceInfo[link.sourceName] || {})}`"
                  :val="getProgressBarValue(link, sourceInfo[link.sourceName] || {})"
                  :val-background="getProgressBarReserved(sourceInfo[link.sourceName] || {})"
                  :max="getProgressBarMax(link, sourceInfo[link.sourceName] || {})"
                  :status="getProgressBarStatus(link, sourceInfo[link.sourceName] || {})"
                  unit="bytes">
                </ProgressBar>
              </div>
            </a>

            <!-- Non-source links: tool and custom links with simple icon style -->
            <a v-else :aria-label="link.name" :href="getLinkHref(link)" class="action button sidebar-link-button"
              :class="{ active: isLinkActive(link) }" @click.prevent="handleLinkClick(link)">
              <div  class="link-container">
                <i v-if="link.icon" :class="`${getIconClass(link.icon)} link-icon`">{{ link.icon }}</i>
                <span>{{ link.name }}</span>
              </div>
            </a>
          </template>
        </transition-group>
    </div>
  </div>
</template>

<script>
import HelpTooltipIcon from "@/components/HelpTooltipIcon.vue";
import { state, getters, mutations } from "@/store";
import {
  hideInteractiveTooltip,
  onComponentTooltipClick,
  onComponentTooltipTouchEnd,
  showHoverComponentTooltip,
  showHoverTooltip,
  useTapForTooltip,
} from "@/utils/tooltipHelp.js";
import ProgressBar from "@/components/ProgressBar.vue";
import { goToItem } from "@/utils/url";
import { getIconClass } from "@/utils/material-symbols";
import { baseSidebarCategory, isRootOnlySidebarCategory, isSourceSidebarCategory } from "@/utils/sidebarCategory";
import { getObjectProperty } from '@/utils/object.js';
import IndexInfo from "@/components/files/IndexInfo.vue";
import { globalVars } from "@/utils/constants";
import { availableTools, hasToolAccess, toolIdFromPath } from "@/utils/toolAccess";
import { showShareDownloadPrompt } from "@/utils/download.js";
import ShareInfo from "@/components/files/ShareInfo.vue";
import FileTree from '@/components/files/FileTree.vue';
import ExpandDropdown from "@/components/settings/ExpandDropdown.vue";
export default {
  name: "SidebarLinks",
  components: {
    HelpTooltipIcon,
    ProgressBar,
    ShareInfo,
    FileTree,
    ExpandDropdown,
  },
  data() {
    return {
      pressedSourceInfo: null,
    };
  },
  computed: {
    useTapForSourceTooltip() {
      return useTapForTooltip();
    },
    editShareText() {
      return this.$t("general.shareManagement");
    },
    canEdit() {
      return state.shareInfo?.canEditShare || false;
    },
    isLoggedIn() {
      return getters.isLoggedIn();
    },
    currentSource() {
      return state.req?.source || null;
    },
    currentPath() {
      return state.req?.path || '/';
    },
    isShare: () => getters.isShare(),
    hasLinks() {
      return this.sidebarLinksToDisplay.length > 0;
    },
    user: () => (state.user || {username: 'anonymous'}),
    sourceInfo() {
      // Access state.sources.info to create reactive dependency
      return state.sources.info;
    },
    activeSource: () => state.sources.current,
    realtimeActive: () => state.realtimeActive,
    hasSourceInfo: () => state.sources.hasSourceInfo,
    showSidebarLinks() {
      // Always show sidebar links section (replaces sources)
      return true;
    },
    hasCustomLinks() {
      // Check if user has customized their links
      return this.user?.sidebarLinks && this.user.sidebarLinks.length > 0;
    },
    // Share info card props
    disableShareCard() {
      return state.shareInfo?.disableShareCard;
    },
    hash() {
      return state.shareInfo?.hash || this.$route.params.hash || "";
    },
    token() {
      return state.shareInfo?.token || this.$route.query.token || "";
    },
    subPath() {
      return state.shareInfo?.path || "/";
    },
    /** Storage sources go last so their usage meter sits at the bottom (Drive); order within each group is kept. */
    orderedSidebarLinks() {
      const links = this.sidebarLinksToDisplay;
      return [
        ...links.filter((link) => !this.isSourceCategory(link.category)),
        ...links.filter((link) => this.isSourceCategory(link.category)),
      ];
    },
    showTreeRoot() {
      return this.isShare || this.sourceNames.length <= 1;
    },
    treeRootLabel() {
      if (this.isShare) {
        return state.shareInfo?.title || this.$t("general.home");
      }
      return this.currentSource || this.$t("general.home");
    },
    treeRootTo() {
      if (this.isShare) {
        return getters.sharePathBase();
      }
      return `/files/${encodeURIComponent(this.currentSource || "")}/`;
    },
    isAtTreeRoot() {
      return getters.currentView() === "listingView" && (this.currentPath === "/" || this.currentPath === "");
    },
    treeRootStatusClass() {
      const status = this.sourceInfo[this.currentSource]?.status;
      return {
        active: this.realtimeActive,
        ready: status === "ready",
        warning: status === "indexing",
        danger: status !== "ready" && status !== "indexing",
      };
    },
    sidebarLinksToDisplay() {
      // If viewing a share, use share's links
      if (getters.isShare() && state.shareInfo?.sidebarLinks?.length > 0) {
        return state.shareInfo.sidebarLinks;
      }
      // If user has custom links, use those
      if (this.hasCustomLinks) {
        return this.user.sidebarLinks;
      }
      // Otherwise, return default links (sources)
      return this.getDefaultLinks();
    },
    // List all source names for the dropdown menu
    sourceNames() {
      return Object.keys(state.sources.info || {});
    },
    // Build a map from source name to its custom link (if any)
    sourceLinkMap() {
      const map = {};
      if (this.user?.sidebarLinks) {
        for (const link of this.user.sidebarLinks) {
          if (this.isSourceCategory(link.category) && link.sourceName && !map[link.sourceName]) {
            map[link.sourceName] = link;
          }
        }
      }
      return map;
    },
    dropdownSourceItems() {
      return this.sourceNames.map(rawName => {
        const customLink = getObjectProperty(this.sourceLinkMap, rawName);
        return {
          rawName,
          displayName: customLink ? customLink.name : rawName,
        };
      });
    },
    sourceDropdownOptions() {
      return this.dropdownSourceItems.map((item) => ({
        value: item.rawName,
        label: item.displayName,
      }));
    },
  },
  beforeUnmount() {
    hideInteractiveTooltip(true);
  },
  methods: {
    isSourceCategory(category) {
      return isSourceSidebarCategory(category);
    },
    isHybridCategory(category) {
      const base = baseSidebarCategory(category);
      return base === 'source-hybrid' || base === 'source-hybrid-2';
    },
    baseSidebarCategory,
    /** Disk capacity for the link's scope: aggregated nested mounts (default) or root filesystem only (-root categories). */
    getUsageTotal(link, sourceInfo) {
      const root = isRootOnlySidebarCategory(link.category);
      const total = root ? (sourceInfo.totalRoot || 0) : (sourceInfo.total || 0);
      return total > 0 ? total : (sourceInfo.total || 1);
    },
    /** Disk-reported used bytes for the link's scope. */
    getDiskUsed(link, sourceInfo) {
      return isRootOnlySidebarCategory(link.category)
        ? (sourceInfo.usedAltRoot || 0)
        : (sourceInfo.usedAlt || 0);
    },
    getIconClass,
    hasUsageInfo(link) {
      if (!this.isSourceCategory(link.category) || !link.sourceName) return false;
      if (!this.hasSourceInfo || !this.isLinkAccessible(link)) return false;
      if (baseSidebarCategory(link.category) === 'source-minimal') return false;
      const info = this.sourceInfo[link.sourceName] || {};
      if (info.scopeQuota?.limitBytes > 0) return true;
      return (info.used || 0) > 0 || (info.usedAlt || 0) > 0;
    },
    getLinkHref(link) {
      // Add baseURL to target for href display
      if (!link.target) return '#';
      const lowerTarget = link.target.toLowerCase();
      if (lowerTarget.startsWith('http://') || lowerTarget.startsWith('https://')) return link.target;

      const baseURL = globalVars.baseURL || '';
      let fullPath;

      // Construct full path based on link category
      if (this.isSourceCategory(link.category)) {
        // For source links, use sourceName and relative target
        if (!link.sourceName) return '#';
        const sourceInfo = this.sourceInfo[link.sourceName];
        if (!sourceInfo) return '#'; // Source not found
        const encodedSourceName = encodeURIComponent(link.sourceName);
        const targetPath = link.target.startsWith('/') ? link.target.substring(1) : link.target;
        fullPath = `/files/${encodedSourceName}/${targetPath}`;
      } else {
        // For other links (tools, custom, share), use target as-is
        fullPath = link.target;
      }

      const target = fullPath.startsWith('/') ? fullPath.substring(1) : fullPath;
      return baseURL + target;
    },
    goHome() {
      if (!this.isLoggedIn) {
        return;
      }
      this.$router.push('/');
    },
    getDefaultLinks() {
      // Generate default links from sources the user can access.
      const defaultLinks = [];
      const scopedSourceNames = new Set(
        (this.user?.scopes || [])
          .map((scope) => scope?.name)
          .filter((name) => typeof name === "string" && name.length > 0),
      );
      const limitToScopes = scopedSourceNames.size > 0;

      if (this.sourceInfo) {
        for (const sourceName of Object.keys(this.sourceInfo)) {
          if (limitToScopes && !scopedSourceNames.has(sourceName)) {
            continue;
          }
          defaultLinks.push({
            name: sourceName,
            category: "source",
            target: "/",
            icon: "",
            sourceName: sourceName,
          });
        }
      }

      return defaultLinks;
    },
    isLinkAccessible(link) {
      // Check if link is accessible
      if (this.isSourceCategory(link.category)) {
        // Use sourceName to check if the source is accessible
        if (!link.sourceName) return false;
        for (const [name] of Object.entries(this.sourceInfo || {})) {
          if (name === link.sourceName) {
            return true;
          }
        }
        return false;
      }
      // Tools and custom links: tools require access; hub requires any tool access
      if (link.category === "tool") {
        const targetPath = String(link.target || "").split(/[?#]/)[0];
        if (targetPath === "/tools") {
          return availableTools().length > 0;
        }
        const toolId = toolIdFromPath(targetPath);
        return toolId ? hasToolAccess(toolId) : false;
      }
      return true;
    },
    isLinkActive(link) {
      // Check if the current route matches this link
      if (this.isSourceCategory(link.category)) {
        // Use sourceName to check if we're currently in this source
        return link.sourceName && state.req.source === link.sourceName;
      }
      // For all other links (tools, custom, share), compare target with route path
      return this.$route.path === link.target;
    },
    getSourceInfoForLink(link) {
      // Method that directly accesses reactive sourceInfo
      // Vue will track this dependency when called in template
      if (!this.isSourceCategory(link.category) || !link.sourceName) return {};
      // Direct access to reactive computed property ensures Vue tracks changes
      return this.sourceInfo && link.sourceName ? this.sourceInfo[link.sourceName] || {} : {};
    },
    getProgressBarStatus(link, sourceInfo) {
      if (sourceInfo.status === 'indexing' && sourceInfo.complexity === 0) {
        return 'indexing';
      }
      if (this.hasUsageScopeMismatch(link, sourceInfo)) {
        return 'conflict';
      }
      return 'default';
    },
    hasUsageScopeMismatch(link, sourceInfo) {
      if (!link || !sourceInfo) return false;
      const base = baseSidebarCategory(link.category);
      if (base !== 'source' && base !== 'source-hybrid' && base !== 'source-hybrid-2') {
        return false;
      }
      if (sourceInfo.usageScopeMismatch && !isRootOnlySidebarCategory(link.category)) {
        return true;
      }
      const used = Number(sourceInfo.used) || 0;
      const total = Number(this.getUsageTotal(link, sourceInfo)) || 0;
      return total > 0 && used > total;
    },
    getProgressBarValue(link, sourceInfo) {
      if (sourceInfo.scopeQuota?.limitBytes > 0) {
        return sourceInfo.scopeQuota.usedBytes || 0;
      }
      // Called with (link, sourceInfo) from both modes
      if (baseSidebarCategory(link.category) === 'source-alt') {
        return this.getDiskUsed(link, sourceInfo);
      }
      return sourceInfo.used || 0;
    },
    getProgressBarMax(link, sourceInfo) {
      if (sourceInfo.scopeQuota?.limitBytes > 0) {
        return sourceInfo.scopeQuota.limitBytes;
      }
      return sourceInfo.total || 1;
    },
    getProgressBarReserved(sourceInfo) {
      return sourceInfo.scopeQuota?.reservedBytes || 0;
    },
    handleLinkClick(link) {
      // Handle special share actions
      if (link.category === 'shareInfo') {
        mutations.showPrompt({ name: "ShareInfo" });
        return;
      }
      if (link.category === 'download') {
        this.goToDownload();
        return;
      }

      // Don't navigate if link is not accessible
      if (!this.isLinkAccessible(link)) {
        return;
      }

      if (this.isSourceCategory(link.category)) {
        // For source links, use sourceName and target (relative path)
        if (!link.sourceName) return;
        const path = link.target || "/";
        goToItem(link.sourceName, path, {}, false, false);
        return;
      }

      const lowerTarget = link.target.toLowerCase();
      if (lowerTarget.startsWith('http://') || lowerTarget.startsWith('https://')) {
        window.open(link.target, "_blank");
        return;
      }
      // For all other links (tools, custom, share), navigate using target directly
      const currentBase = this.$route.path;
      const targetBase = link.target.split('?')[0]; // strip any query from target -- this to avoid an error when navigating back with browser history

      if (currentBase === targetBase) {
        this.$router.replace(link.target);
      } else {
        this.$router.push(link.target);
      }
      mutations.closeTopPrompt();
    },
    goToDownload() {
      showShareDownloadPrompt([{
        path: state.req?.path || "/",
        name: state.req?.name,
        type: state.req?.type,
        size: state.req?.size,
        isDir: state.req?.type === "directory",
      }]);
    },
    navigateTo(path, hash) {
      mutations.setPreviousHistoryItem({
        name: state.req?.name,
        source: state.req?.source,
        path: state.req?.path,
      });
      this.$router.push({ path: path, hash: hash });
      mutations.closeTopPrompt();
    },
    openSidebarLinksPrompt() {
      if (!this.isLoggedIn) {
        return;
      }
      mutations.showPrompt({
        name: "SidebarLinks",
      });
    },
    beforeEnter(el) {
      el.style.height = '0';
      el.style.opacity = '0';
    },
    enter(el, done) {
      el.style.transition = '';
      el.style.height = '0';
      el.style.opacity = '0';
      // Force reflow
      void el.offsetHeight;
      el.style.transition = 'height 0.3s, opacity 0.3s';
      el.style.height = `${el.scrollHeight}px`;
      el.style.opacity = '1';
      setTimeout(() => {
        el.style.height = 'auto';
        done();
      }, 300);
    },
    leave(el, done) {
      el.style.transition = 'height 0.3s, opacity 0.3s';
      el.style.height = `${el.scrollHeight}px`;
      void el.offsetHeight;
      el.style.height = '0';
      el.style.opacity = '0';
      setTimeout(done, 300);
    },
    showTooltip(event, text) {
      if (text) {
        showHoverTooltip(text, event);
      }
    },
    hideTooltip() {
      hideInteractiveTooltip();
    },
    onSourceInfoClick(event, info) {
      onComponentTooltipClick(this.sourceInfoTooltipOptions(event, info));
    },
    onSourceInfoTouchStart(sourceName) {
      if (useTapForTooltip()) {
        this.pressedSourceInfo = sourceName;
      }
    },
    onSourceInfoTouchEnd(event, info) {
      this.pressedSourceInfo = null;
      onComponentTooltipTouchEnd(this.sourceInfoTooltipOptions(event, info));
    },
    onSourceInfoTouchCancel() {
      this.pressedSourceInfo = null;
    },
    sourceInfoTooltipOptions(event, info) {
      return {
        event,
        component: IndexInfo,
        componentProps: { info },
        pointerEvents: this.useTapForSourceTooltip,
        propsMatcher: (props) => props.info?.name === info?.name,
      };
    },
    onSourceInfoMouseEnter(event, info) {
      this.showSourceHoverTooltip(event, info);
    },
    onSourceInfoMouseLeave() {
      this.pressedSourceInfo = null;
      if (this.useTapForSourceTooltip) {
        return;
      }
      this.hideTooltip();
    },
    showSourceHoverTooltip(event, info) {
      if (!info?.name) {
        return;
      }
      showHoverComponentTooltip({
        component: IndexInfo,
        componentProps: { info },
        event,
      });
    },
    async showEditShareHover() {
      if (!this.canEdit) {
        return;
      }
      // Get the current share hash and fetch full share details
      const shareHash = state.shareInfo?.hash;
      if (!shareHash) {
        console.error("No share hash found");
        return;
      }

      try {
        // Fetch the full share details to pass to the edit dialog
        // The shareInfo object should already have most details we need
        const shareData = {
          ...state.shareInfo,
          hash: shareHash,
        };

        mutations.showPrompt({
          name: "share",
          props: {
            editing: true,
            link: shareData,
          },
        });
      } catch (err) {
        console.error("Failed to open edit share dialog:", err);
      }
    },
    navigateToSource(sourceName) {
      if (!sourceName || sourceName === this.activeSource) {
        return;
      }
      goToItem(sourceName, '/', {}, false, false);
    },
  },
};
</script>

<style scoped>
.sidebar-links {
  display: flex;
  flex-direction: column;
  flex: 1 1 auto;
  min-height: 0;
  padding: 0;
  overflow: hidden;
  background: transparent;
  box-shadow: none;
}

/* Tree takes the free height and scrolls; links stay pinned underneath. */
.sidebar-tree-section {
  flex: 1 1 auto;
  min-height: 0;
  overflow: auto;
  scrollbar-width: thin;
  padding-bottom: 8px;
}

.sidebar-links-section {
  flex: 0 0 auto;
  max-height: 45%;
  overflow: auto;
  scrollbar-width: thin;
  padding: 8px 0;
  border-top: 1px solid var(--divider);
}

.tree-root {
  display: flex;
  align-items: center;
  gap: 8px;
  height: 32px;
  padding: 0 12px 0 8px;
  border-radius: var(--radius-full);
  color: var(--textPrimary);
  font-size: 14px;
  font-weight: 600;
  text-decoration: none;
  transition: background-color 0.1s ease;
}

.tree-root span {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.tree-root:hover {
  background: var(--state-hover);
}

.tree-root.active {
  background: var(--accent-container);
  color: var(--on-accent-container);
}

.tree-root__icon {
  font-size: 20px;
  color: var(--folder-color);
}

.nav-section-label {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 32px;
  padding: 0 4px 0 16px;
  color: var(--textSecondary);
  font-family: var(--font-display);
  font-size: 12px;
  font-weight: 500;
  letter-spacing: 0.02em;
}

.nav-section-action {
  width: 32px;
  height: 32px;
  min-width: 32px;
  min-height: 32px;
  padding: 0;
  border-radius: var(--radius-full);
  background: transparent;
  color: var(--textSecondary);
  cursor: pointer;
  opacity: 0;
  transition: opacity 0.15s ease, background-color 0.15s ease;
}

.nav-section-action:hover {
  background: var(--state-hover);
}

.nav-section-action:focus-visible,
.sidebar-links-section:hover .nav-section-action {
  opacity: 1;
}

.nav-section-action .material-symbols {
  font-size: 18px;
}

.file-tree-container {
  margin-top: 0;
}

.share-info-section {
  margin: 0 0 8px;
  padding-bottom: 8px;
  border-bottom: 1px solid var(--divider);
}

.sidebar-links .inner-card {
  display: flex;
  flex-direction: column;
  align-items: stretch;
  width: 100%;
  gap: 2px;
}

/* Nav rows: flat pill, soft accent when active (Drive / M3 navigation drawer). */
.sidebar-link-button,
.source-button {
  display: block;
  width: 100%;
  max-width: 100%;
  margin: 0;
  padding: 0 16px;
  border-radius: var(--radius-full);
  font-weight: 400;
  background: transparent;
  box-shadow: none !important;
  color: var(--textPrimary);
  text-decoration: none;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.sidebar-link-button:hover {
  background: var(--state-hover);
  transform: none !important;
}

.sidebar-link-button.active {
  background: var(--accent-container);
  color: var(--on-accent-container);
  font-weight: 600;
}

/* Source rows: the pill is the 36px row only; the usage meter sits under it, unhighlighted. */
.source-button {
  padding: 0;
  background: transparent !important;
}

.source-button:hover {
  transform: none !important;
}

.link-container,
.source-container {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 12px;
  min-height: 36px;
  color: var(--textPrimary);
  font-size: 14px;
}

.source-container {
  padding: 0 16px;
  border-radius: var(--radius-full);
  transition: background-color 0.15s ease;
}

.source-button:hover .source-container {
  background: var(--state-hover);
}

.source-button.active .source-container {
  background: var(--accent-container);
  font-weight: 600;
}

.source-container span,
.link-container span {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.link-icon,
.source-container > i:first-child {
  width: 20px;
  flex: 0 0 20px;
  padding: 0 !important;
  font-size: 20px;
  color: var(--textSecondary);
}

.active .link-icon,
.active .source-container > i:first-child {
  color: var(--on-accent-container);
}

.source-button.disabled {
  opacity: 0.5;
  cursor: not-allowed;
  pointer-events: none;
}

.sidebar-link-button.active .link-container,
.source-button.active .source-container {
  color: var(--on-accent-container);
}

/* Usage meter: thin track with the label underneath, aligned with the row label. */
.usage-info {
  padding: 4px 16px 8px 48px;
}

.usage-info :deep(.vue-simple-progress) {
  height: 4px !important;
  min-height: 4px !important;
  margin: 0 0 22px !important;
  border: 0 !important;
  border-radius: var(--radius-full) !important;
  background: var(--divider) !important;
  overflow: visible !important;
}

.usage-info :deep(.vue-simple-progress-bar) {
  height: 4px !important;
  border-radius: var(--radius-full) !important;
}

.usage-info :deep(.vue-simple-progress-bar:not(.vue-simple-progress-bar-background)) {
  background: var(--accent) !important;
}

/* ProgressBar centres its label on the bar with inline top:50% + translateY(-50%);
   in the sidebar the label sits under the 4px track instead, independent of font size. */
.usage-info :deep(.vue-simple-progress-text) {
  inset: 100% auto auto 0 !important;
  width: auto !important;
  margin-top: 4px;
  padding: 0 !important;
  transform: none !important;
  line-height: 16px;
  color: var(--textSecondary) !important;
  font-size: 12px !important;
  text-align: left !important;
  white-space: nowrap;
}

.tooltip-info-icon {
  font-size: 16px;
  color: var(--textSecondary);
  opacity: 0;
  transition: opacity 0.15s ease;
}

.source-button:hover .tooltip-info-icon {
  opacity: 1;
}

.expand-enter-active,
.expand-leave-active {
  transition: height 0.3s cubic-bezier(0.4, 0, 0.2, 1), opacity 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  overflow: hidden;
  will-change: opacity, max-height;
}

.expand-enter,
.expand-leave-to {
  height: 0 !important;
  opacity: 0;
}

/* Source status: small dot instead of the large pulse. */
.realtime-pulse {
  width: 20px;
  height: 20px;
  flex: 0 0 20px;
  margin: 0;
}

.realtime-pulse > .center {
  r: 4px;
}

.realtime-pulse > .pulse {
  display: none;
  fill-opacity: 0;
  transform-origin: 50% 50%;
  animation: pulse 10s infinite backwards;
}

.realtime-pulse.active > .pulse {
  display: block;
}

.realtime-pulse.ready > .pulse {
  fill: #1e8e3e;
  stroke: #1e8e3e;
}

.realtime-pulse.danger > .pulse {
  fill: rgb(190 147 147);
  stroke: #d93025;
}

.realtime-pulse.warning > .pulse {
  fill: #f9ab00;
  stroke: #f9ab00;
}

.realtime-pulse.ready > .center {
  fill: #1e8e3e;
}

.realtime-pulse.danger > .center {
  fill: #d93025;
}

.realtime-pulse.warning > .center {
  fill: #f9ab00;
}

@keyframes pulse {
  from {
    stroke-width: 3px;
    stroke-opacity: 1;
    transform: scale(0.3);
  }

  to {
    stroke-width: 0;
    stroke-opacity: 0;
    transform: scale(1.5);
  }
}

.source-card-container {
  position: relative;
  margin: 0 0 4px;
  padding: 0 8px;
}

.sidebar-source-switcher {
  margin: 0;
}

.sidebar-divider-container {
  width: 100%;
  margin: 8px 0;
}

.sidebar-divider {
  width: 100%;
  margin: 0;
  border: none;
  border-top: 1px solid var(--divider);
}

.sidebar-divider-text {
  display: flex;
  align-items: center;
  height: 32px;
  padding-left: 16px;
  color: var(--textSecondary);
  font-size: 12px;
  font-weight: 500;
}
</style>
