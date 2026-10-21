<template>
  <div class="listing-item-header" :class="{ 'desktop-view': !isMobile && !gridMode, 'grid-mode': gridMode }">
    <p
      :class="{ active: nameSorted }"
      class="name"
      role="button"
      tabindex="0"
      @click="sort('name')"
      @keydown.enter.prevent="sort('name')"
      @keydown.space.prevent="sort('name')"
      :title="$t('files.sortByName')"
      :aria-label="$t('files.sortByName')"
    >
      <span>{{ $t("general.name") }}</span>
      <i class="material-symbols">{{ nameIcon }}</i>
    </p>

    <p
      :class="{ active: sizeSorted }"
      class="size"
      role="button"
      tabindex="0"
      @click="sort('size')"
      @keydown.enter.prevent="sort('size')"
      @keydown.space.prevent="sort('size')"
      :title="$t('files.sortBySize')"
      :aria-label="$t('files.sortBySize')"
    >
      <i class="material-symbols">{{ sizeIcon }}</i>
      <span>{{ $t("general.size") }}</span>
    </p>

    <p
      :class="{ active: modifiedSorted }"
      class="modified"
      role="button"
      tabindex="0"
      @click="sort('modified')"
      @keydown.enter.prevent="sort('modified')"
      @keydown.space.prevent="sort('modified')"
      :title="$t('files.sortByLastModified')"
      :aria-label="$t('files.sortByLastModified')"
    >
      <i class="material-symbols">{{ modifiedIcon }}</i>
      <span>{{ $t("files.lastModified") }}</span>
    </p>

    <p
      v-if="showKindColumn"
      :class="{ active: kindSorted }"
      class="kind"
      role="button"
      tabindex="0"
      @click="sort('kind')"
      @keydown.enter.prevent.stop="sort('kind')"
      @keydown.space.prevent.stop="sort('kind')"
      :title="$t('files.sortByType')"
      :aria-label="$t('files.sortByType')"
    >
      <span>{{ $t("general.type") }}</span>
      <i class="material-symbols">{{ kindIcon }}</i>
    </p>

    <p
      v-if="showCreatedColumn"
      :class="{ active: createdSorted }"
      class="created"
      role="button"
      tabindex="0"
      @click="sort('created')"
      @keydown.enter.prevent.stop="sort('created')"
      @keydown.space.prevent.stop="sort('created')"
      :title="$t('files.sortByCreationTime')"
      :aria-label="$t('files.sortByCreationTime')"
    >
      <i class="material-symbols">{{ createdIcon }}</i>
      <span>{{ $t("files.creationTime") }}</span>
    </p>

    <p
      v-if="hasDuration"
      :class="{ active: durationSorted }"
      class="duration"
      role="button"
      tabindex="0"
      @click="sort('duration')"
      @keydown.enter.prevent="sort('duration')"
      @keydown.space.prevent="sort('duration')"
      :title="$t('files.sortByDuration')"
      :aria-label="$t('files.sortByDuration')"
    >
      <i class="material-symbols">{{ durationIcon }}</i>
      <span>{{ $t("files.duration") }}</span>
    </p>
    <span v-if="quickDownloadEnabled" class="placeholder"></span>
  </div>
</template>

<script>
import { state, getters, mutations } from "@/store";

export default {
  name: "ListingHeader",
  props: {
    hasDuration: {
      type: Boolean,
      default: false,
    },
    /** When true, sort via pickerSorting (destination pickers) instead of the main listing sort. */
    usePickerSorting: {
      type: Boolean,
      default: false,
    },
    useNavigationSorting: {
      type: Boolean,
      default: false,
    },
  },
  computed: {
    isMobile() {
      return getters.isMobile();
    },
    sortConfig() {
      if (this.useNavigationSorting) return getters.navigationSorting();
      return this.usePickerSorting ? getters.pickerSorting() : getters.sorting();
    },
    nameSorted() {
      return this.sortConfig.by === "name";
    },
    sizeSorted() {
      return this.sortConfig.by === "size";
    },
    modifiedSorted() {
      return this.sortConfig.by === "modified";
    },
    createdSorted() {
      return this.sortConfig.by === "created";
    },
    kindSorted() {
      return this.sortConfig.by === "kind";
    },
    durationSorted() {
      return this.sortConfig.by === "duration";
    },
    ascOrdered() {
      return this.sortConfig.asc;
    },
    galleryView() {
      return getters.viewMode() === "gallery";
    },
    gridMode() {
      // Grids and phones get sort chips; desktop lists get a column header.
      return (!this.isListMode || this.isMobile) && !this.usePickerSorting && !this.useNavigationSorting;
    },
    isListMode() {
      const mode = getters.viewMode();
      return mode === "list" || mode === "compact";
    },
    showKindColumn() {
      return this.isListMode && !this.usePickerSorting && !this.useNavigationSorting && state.user?.showTypeColumn;
    },
    showCreatedColumn() {
      return this.isListMode && !this.usePickerSorting && !this.useNavigationSorting && state.user?.showCreationDateColumn;
    },
    nameIcon() {
      if (this.nameSorted && !this.ascOrdered) {
        return "arrow_upward";
      }
      return "arrow_downward";
    },
    sizeIcon() {
      if (this.sizeSorted && this.ascOrdered) {
        return "arrow_downward";
      }
      return "arrow_upward";
    },
    modifiedIcon() {
      if (this.modifiedSorted && this.ascOrdered) {
        return "arrow_downward";
      }
      return "arrow_upward";
    },
    createdIcon() {
      if (this.createdSorted && this.ascOrdered) {
        return "arrow_downward";
      }
      return "arrow_upward";
    },
    kindIcon() {
      if (this.kindSorted && this.ascOrdered) {
        return "arrow_downward";
      }
      return "arrow_upward";
    },
    durationIcon() {
      if (this.durationSorted && this.ascOrdered) {
        return "arrow_downward";
      }
      return "arrow_upward";
    },
    quickDownloadEnabled() {
      if (this.usePickerSorting || this.useNavigationSorting) {
        return false;
      }
      if (getters.isMobile()) {
        return false
      }
      if (getters.isShare()) {
        return state.shareInfo?.quickDownload;
      }
      return state.user?.quickDownload && !this.galleryView;
    },
  },
  methods: {
    sort(field) {
      let asc = false;
      if (
        (field === "name" && this.nameIcon === "arrow_upward") ||
        (field === "size" && this.sizeIcon === "arrow_upward") ||
        (field === "modified" && this.modifiedIcon === "arrow_upward") ||
        (field === "created" && this.createdIcon === "arrow_upward") ||
        (field === "kind" && this.kindIcon === "arrow_upward") ||
        (field === "duration" && this.durationIcon === "arrow_upward")
      ) {
        asc = true;
      }
      // Commit the updateSort mutation
      if (this.useNavigationSorting) {
        mutations.updateNavigationSortConfig({ field, asc });
        return;
      }
      if (this.usePickerSorting) {
        mutations.updatePickerSortConfig({ field, asc });
        return;
      }
      mutations.updateListingSortConfig({ field, asc });
      mutations.updateListingItems();
    },
  },
};
</script>

<style scoped>
/* Table header for list/compact rows (Drive "Name | Last modified | File size"). */
.listing-item-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  height: 40px;
  box-sizing: border-box;
  margin-bottom: 0 !important;
  padding: 0 8px;
  border-bottom: 1px solid var(--divider);
  background: var(--surfacePrimary);
  color: var(--textPrimary);
  font-size: 14px;
  font-weight: 500;
  user-select: none;
  z-index: 999;
}

p {
  display: flex;
  align-items: center;
  gap: 4px;
  width: 100%;
  height: 32px;
  margin: 0;
  box-sizing: border-box;
  cursor: pointer;
}

span {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.name {
  flex: 1;
  min-width: 0;
}

.size,
.modified,
.duration {
  flex: 1;
  justify-content: flex-end;
  text-align: end;
}

/* Column widths are shared with the list/compact rows (see listing.css) so the
   header lines up with the items and the name column gets the remaining space. */
/* stylelint-disable no-unknown-custom-properties -- defined in css/listing.css */
.desktop-view {
  padding-right: calc(8px + var(--listing-col-gap));
}

/* With quick download the rows end in a ~2.4rem download icon after the inset */
.desktop-view:has(> .placeholder) {
  padding-right: 8px;
}

.desktop-view > .placeholder {
  flex: 0 0 auto;
  order: 7;
  width: calc(var(--listing-col-gap) + 2.4rem);
}

.desktop-view .size,
.desktop-view .modified,
.desktop-view .kind,
.desktop-view .created,
.desktop-view .duration {
  flex: 0 0 auto;
  padding-left: var(--listing-col-gap);
  justify-content: flex-start;
  text-align: left;
}

/* Drive column order: name, modified, size */
.desktop-view .modified {
  order: 2;
  width: var(--listing-col-modified);
}

.desktop-view .size {
  order: 3;
  width: var(--listing-col-size);
}

.desktop-view .kind {
  order: 4;
  width: var(--listing-col-kind);
}

.desktop-view .created {
  order: 5;
  width: var(--listing-col-created);
}

.desktop-view .duration {
  order: 6;
  width: var(--listing-col-duration);
}

/* stylelint-enable no-unknown-custom-properties */

/* Sort arrows sit after the label and appear on hover or for the active column. */
i {
  flex-shrink: 0;
  order: 2;
  font-size: 18px;
  opacity: 0;
  transition: opacity 0.1s ease;
}

.active i,
p:hover i {
  opacity: 1;
}

/* Grid views: sort controls as a row of chips, right-aligned. */
.grid-mode {
  justify-content: flex-end;
  gap: 8px;
  height: 48px;
  border-bottom: 0;
}

.grid-mode p {
  flex: 0 0 auto;
  width: auto;
  padding: 0 12px;
  border: 1px solid var(--divider);
  border-radius: var(--radius-s);
  color: var(--textSecondary);
  font-size: 13px;
}

.grid-mode p:hover {
  background: var(--state-hover);
}

.grid-mode p.active {
  border-color: transparent;
  background: var(--accent-container);
  color: var(--on-accent-container);
}

.grid-mode .placeholder {
  display: none;
}
</style>
