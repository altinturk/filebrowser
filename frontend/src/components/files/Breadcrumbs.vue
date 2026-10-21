<template>
  <div id="breadcrumbs">
    <ul>
      <li>
        <router-link :to="base" :aria-label="$t('general.home')" :title="rootLabel"
          :class="{ 'droppable-breadcrumb': isDroppable, 'drag-over': dragOverItem?.type === 'home', current: items.length === 0 }"
          @dragenter.prevent="dragEnter($event, homeLink)"
          @dragleave.prevent="dragLeave($event, homeLink)"
          @dragover.prevent="dragOver($event, homeLink)"
          @drop.prevent="drop($event, homeLink)">
          <span class="breadcrumb-text">{{ rootLabel }}</span>
        </router-link>
      </li>
      <li class="item" v-for="(link, index) in items" :key="index">
        <i class="material-symbols breadcrumb-sep" aria-hidden="true">chevron_right</i> <!-- eslint-disable-line @intlify/vue-i18n/no-raw-text -->
        <router-link
          :to="link.url"
          :aria-label="`breadcrumb-link-${link.name}`"
          :title="link.name"
          :key="index"
          :class="{ changeAvailable: hasUpdate, current: index === items.length - 1,
            'droppable-breadcrumb': isDroppable && link.type !== 'truncated',
            'drag-over': dragOverItem?.path === link.path, }"
          @dragenter="dragEnter($event, link)"
          @dragleave="dragLeave($event, link)"
          @dragover="dragOver($event, link)"
          @drop="drop($event, link)">
          <span class="breadcrumb-text">{{ link.name }}</span>
        </router-link>
      </li>
    </ul>
  </div>
</template>

<script>
import { resourcesApi } from "@/api";
import { notify } from "@/notify";
import { getters, mutations, state } from "@/store";
import { url } from "@/utils";
import { getObjectProperty } from '@/utils/object.js';
import { goToItemNotificationButton } from "@/utils/notificationActions";
import { notifyMoveCopyFailure } from "@/utils/appNotifications";

export default {
  name: "breadcrumbs",
  data() {
    return {
      base: "/files/",
      dragOverItem: null, // Track which breadcrumb is being dragged over
    };
  },
  props: ["noLink"],
  mounted() {
    this.updatePaths();
    window.addEventListener('dragend', this.clearDragState);
  },
  beforeUnmount() {
    window.removeEventListener('dragend', this.clearDragState);
  },
  watch: {
    $route() {
      this.updatePaths();
    },
    req() {
      this.updatePaths();
    },
  },
  computed: {
    req() {
      return state.req;
    },
    hasUpdate() {
      return state.req.hasUpdate;
    },
    isDroppable() {
      return getters.sourcePermissions().modify
    },
    rootLabel() {
      if (getters.isShare()) {
        return state.shareInfo?.title || this.$t("general.home");
      }
      return state.req?.source || this.$t("general.home");
    },
    homeLink() {
      return {
        name: this.$t('general.home'),
        url: this.base,
        path: '/',
        type: 'home',
      };
    },
    scrollRatio() {
      const ratio = state.listing.scrollRatio || 0;
      return ratio;
    },
    items() {
      const req = state.req;
      if (!req.path) return [];
      const encodedPathString = url.encodedPath(state.req.path);
      const originalParts = state.req.path.split("/");
      const encodedParts = encodedPathString.split("/");
      // Remove empty strings from both arrays consistently
      if (originalParts[0] === "") {
        originalParts.shift();
        encodedParts.shift();
      }
      if (originalParts[originalParts.length - 1] === "") {
        originalParts.pop();
        encodedParts.pop();
      }
      const breadcrumbs = [];
      let buildRef = this.base;
      let accumulatedPath = "";

      for (let i = 0; i < originalParts.length; i++) {
        const origPart = getObjectProperty(originalParts, i);
        const encodedElement = getObjectProperty(encodedParts, i);
        buildRef = `${buildRef + encodedElement}/`;
        accumulatedPath = accumulatedPath ? `${accumulatedPath}/${origPart}` : origPart;

        breadcrumbs.push({
          name: origPart,
          url: buildRef,
          path: `/${accumulatedPath}`,
          type: 'normal'
        });
      }
      if (breadcrumbs.length > 3) {
        while (breadcrumbs.length !== 4) {
          breadcrumbs.shift();
        }
        breadcrumbs[0] = {
          name: "...",
          url: breadcrumbs[0].url, // Keep the URL of the first visible breadcrumb
          path: breadcrumbs[0].path, // Same here, but with the path
          type: 'truncated'
        };
      }
      return breadcrumbs;
    },
  },
  methods: {
    updatePaths() {
      if (getters.isShare()) {
        this.base = getters.sharePathBase();
      } else {
        this.base = `/files/${state.req.source}/`;
      }
    },

    dragEnter(event, link) {
      // Don't allow drag over "..." (is a truncated breadcrumb)
      if (!this.isDroppable || link.type === 'truncated') return;
      if (!event.dataTransfer.types.includes("application/x-filebrowser-internal-drag")) return;
      event.preventDefault();
      this.dragOverItem = link;
    },

    dragOver(event, link) {
      if (!this.isDroppable || link.type === 'truncated') return;
      if (!event.dataTransfer.types.includes("application/x-filebrowser-internal-drag")) return;
      event.preventDefault();
    },

    dragLeave(event, link) {
      // Don't clear drag state if we're just moving to a child element
      if (event.currentTarget.contains(event.relatedTarget)) {
        return;
      }

      if (this.dragOverItem?.path === link.path ||
          (this.dragOverItem?.type === 'home' && link.type === 'home')) {
        this.clearDragState();
      }
    },

    clearDragState() {
      this.dragOverItem = null;
    },

    async drop(event, link) {
      // Don't allow drop on "..."
      if (link.type === 'truncated') return;
      if (!this.isDroppable) return;

      event.preventDefault();
      event.stopPropagation();
      this.clearDragState();

      const isInternal = Array.from(event.dataTransfer.types).includes(
        "application/x-filebrowser-internal-drag"
      );

      if (!isInternal) return;

      // Get the target path for this breadcrumb
      const targetPath = link.path; // Use the path from the link object
      const currentPath = state.req.path;
      const source = state.req.source;

      // Normalize paths for comparison
      const normalizePath = (path) => {
        if (!path || path === "/") return "/";
        return path.replace(/\/$/, ''); // Remove trailing slash
      };

      const normalizedTarget = normalizePath(targetPath);
      const normalizedCurrent = normalizePath(currentPath);

      if (normalizedTarget === normalizedCurrent) {
        notify.showErrorToast(this.$t("files.sameFolder"));
        console.error("Cannot move to same folder");
        return;
      }

      // Build list of items to move from selected items
      let itemsToMove = [];
      for (const i of state.selected) {
        if (i < 0 || i >= state.req.length) continue;

        const selectedItem = state.req.items.at(i);

        let fromPath = selectedItem.path;

        if (!fromPath) {
          fromPath = url.joinPath(state.req.path, selectedItem.name);
        }

        itemsToMove.push({
          from: fromPath,
          fromSource: selectedItem.source,
          to: url.joinPath(targetPath, selectedItem.name),
          toSource: source,
          itemType: selectedItem.type,
          name: selectedItem.name
        });
      }

      // Filter out invalid moves
      itemsToMove = itemsToMove.filter(item => {
        if (item.from === item.to) return false;

        // Prevent moving a directory into itself -- likely never will happen but just in case
        if (item.itemType === 'directory') {
          const fromDir = normalizePath(item.from);
          const toDir = normalizePath(item.to);

          // Check if destination is inside the directory
          if (toDir.startsWith(`${fromDir}/`)) {
            return false;
          }
        }
        return true;
      });

      // Check for conflicts in target directory
      let targetDirItems = [];
      try {
        if (getters.isShare()) {
          const response = await resourcesApi.fetchFilesPublic(targetPath, state.shareInfo.hash);
          targetDirItems = response?.items;
        } else {
          const response = await resourcesApi.fetchFiles(source, targetPath);
          targetDirItems = response?.items;
        }
      } catch (_e) {
        notify.showError(this.$t("files.cannotAccesDir"));
        return;
      }

      // if any item conflics will show replace-rename prompt later
      const conflict = itemsToMove.some(item => {
        const itemName = item.to.split('/').pop(); // Extract filename from destination path
        return targetDirItems.some(targetItem => targetItem.name === itemName);
      });

      const moveAction = async (overwrite, rename) => {
        mutations.showPrompt({
          name: "move",
          props: { operationInProgress: true },
        });

        try {
          if (getters.isShare()) {
            await resourcesApi.moveCopyPublic(state.shareInfo.hash, itemsToMove, "move", overwrite, rename);
          } else {
            await resourcesApi.moveCopy(itemsToMove, "move", overwrite, rename);
          }

          notify.showSuccess(this.$t("prompts.moveSuccess"), {
            icon: "folder",
            buttons: [
              goToItemNotificationButton(
                this.$t("buttons.goToItem"),
                source,
                targetPath,
                getters.isShare()
              ),
            ],
          });
          mutations.closeTopPrompt();
          mutations.setReload(true);
        } catch (error) {
          mutations.closeTopPrompt();
          notifyMoveCopyFailure(error);
        }
      };

      if (conflict) {
        mutations.showPrompt({
          name: "replace-rename",
          pinned: true,
          confirm: (event, option) => {
            const overwrite = option === "overwrite";
            const rename = option === "rename";
            event.preventDefault();
            mutations.closeTopPrompt();
            void moveAction(overwrite, rename);
          },
        });
        return;
      }
      try {
        await moveAction(false, false);
      } catch (error) {
        notifyMoveCopyFailure(error);
      }
    },
  },
};
</script>

<style scoped>
#breadcrumbs {
  min-width: 0;
  overflow-x: auto;
  scrollbar-width: none;
}

#breadcrumbs::-webkit-scrollbar {
  display: none;
}

#breadcrumbs ul {
  display: flex;
  align-items: center;
  margin: 0;
  padding: 0;
  list-style: none;
}

#breadcrumbs ul li {
  display: inline-flex;
  align-items: center;
  min-width: 0;
  position: relative;
}

#breadcrumbs ul li a {
  display: inline-flex;
  align-items: center;
  height: 36px;
  max-width: 40vw;
  padding: 0 12px;
  border-radius: var(--radius-full);
  color: var(--textSecondary);
  font-family: var(--font-display);
  font-size: 22px;
  font-weight: 400;
  line-height: 28px;
  white-space: nowrap;
  text-decoration: none;
  user-select: none;
  transition: background-color 0.15s ease, color 0.15s ease;
}

#breadcrumbs ul li a.current {
  color: var(--textPrimary);
}

#breadcrumbs ul li a:hover {
  background: var(--state-hover);
  color: var(--textPrimary);
}

#breadcrumbs .breadcrumb-text {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.breadcrumb-sep {
  flex: 0 0 auto;
  font-size: 24px;
  color: var(--textSecondary);
}

#breadcrumbs ul li a.changeAvailable {
  font-style: italic;
}

.drag-over {
  background: var(--accent-container) !important; /* Needs !important to make the hover effect work when dragging items */
  color: var(--on-accent-container) !important;
  z-index: 2;
}

@media (width <= 768px) {
  #breadcrumbs ul li a {
    font-size: 18px;
    padding: 0 8px;
  }
}
</style>
