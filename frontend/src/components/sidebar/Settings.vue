<template>
  <div v-if="isMobile" role="button" tabindex="0" class="card item clickable settings-card" @click="closeSettings"
    @keydown.enter.prevent="!$event.repeat && closeSettings()" @keydown.space.prevent @keyup.space="closeSettings">
    <span class="settings-item-content">
      <span class="material-symbols-outlined settings-icon">close</span> <!-- eslint-disable-line @intlify/vue-i18n/no-raw-text -->
      {{ $t("general.exit") }}
    </span>
  </div>
  <template v-for="setting in settings" :key="`${setting.id}-sidebar`">
    <div
      v-if="setting.sections"
      class="card item settings-card-collapsible"
      :class="{ hidden: !shouldShow(setting), 'settings-card-collapsible--active': canExpand(setting) && isSectionActive(setting) && expandedSections[setting.id] }"
    >
      <div
        role="button"
        tabindex="0"
        class="settings-card-collapsible-header settings-card clickable"
        :class="{ 'active-settings': isHeaderActive(setting) }"
        @click="headerClick(setting)"
        @keydown.enter.prevent="!$event.repeat && headerClick(setting)"
        @keydown.space.prevent
        @keyup.space="headerClick(setting)"
      >
        <span class="settings-item-content">
          <span class="material-symbols-outlined settings-icon">{{ setting.icon }}</span>
          {{ settingLabel(setting) }}
        </span>
        <i
          v-if="canExpand(setting)"
          class="material-symbols-outlined settings-card-collapsible-chevron"
          :class="{ rotated: expandedSections[setting.id] }"
          :aria-expanded="!!expandedSections[setting.id]"
          role="button"
          tabindex="0"
          @click.stop="expandSection(setting)"
          @keydown.enter.stop.prevent="!$event.repeat && expandSection(setting)"
          @keydown.space.stop.prevent
          @keyup.space.stop="expandSection(setting)"
        >
          keyboard_arrow_down
        </i>
      </div>
      <div class="settings-card-sub-item" :class="{ 'settings-card-collapsible--expanded': expandedSections[setting.id] }">
        <div class="settings-card-sub-item-inner">
          <div
            v-for="section in visibleSections(setting)"
            :key="section.id"
            role="button"
            tabindex="0"
            class="settings-card-collapsible-sub-item settings-card clickable"
            :class="{ 'active-settings': active(`${setting.id}-${section.id}`) }"
            @click.stop="setView(`${setting.id}-${section.id}`)"
            @keydown.enter.stop.prevent="!$event.repeat && setView(`${setting.id}-${section.id}`)"
            @keydown.space.stop.prevent
            @keyup.space.stop="setView(`${setting.id}-${section.id}`)"
          >
            <span class="settings-item-content">
              <span class="material-symbols-outlined settings-icon">{{ section.icon }}</span>
              {{ $t(section.label, { suffix: "" }) }}
            </span>
          </div>
        </div>
      </div>
    </div>
    <div
      v-else
      :id="`${setting.id}-sidebar`"
      role="button"
      tabindex="0"
      class="card item clickable settings-card"
      @click="setView(`${setting.id}-main`)"
      @keydown.enter.prevent="!$event.repeat && setView(`${setting.id}-main`)"
      @keydown.space.prevent
      @keyup.space="setView(`${setting.id}-main`)"
      :class="{
        hidden: !shouldShow(setting),
        'active-settings': active(`${setting.id}-main`),
      }">
      <span v-if="shouldShow(setting)" class="settings-item-content">
        <span class="material-symbols-outlined settings-icon">{{ setting.icon }}</span>
        {{ settingLabel(setting) }}
      </span>
    </div>
  </template>
</template>

<script>
import { state, getters, mutations } from "@/store";
import { getObjectProperty } from '@/utils/object.js';
import { settings } from "@/utils/constants";
import { router } from "@/router";

export default {
  name: "SidebarSettings",
  data() {
    return {
      settings, // Initialize the settings array in data
      expandedSections: {},
    };
  },
  computed: {
    currentHash: () => getters.currentHash(),
    isMobile: () => getters.isMobile(),
    activeView: () => state.activeSettingsView || "",
    expandableSettings() {
      return this.settings.filter((setting) => setting.sections);
    },
    showAdvancedProfile() {
      return !!state.user?.showAdvancedProfile;
    },
  },
  watch: {
    activeView() {
      this.syncExpanded();
    },
    showAdvancedProfile() {
      this.syncExpanded();
    },
  },
  mounted() {
    requestAnimationFrame(() => this.syncExpanded());
  },
  methods: {
    syncExpanded() {
      for (const setting of this.expandableSettings) {
        this.expandedSections[setting.id] = this.isSectionActive(setting) && this.canExpand(setting);
      }
    },
    isHeaderActive(setting) {
      return this.isSectionActive(setting) && (!this.canExpand(setting) || !this.expandedSections[setting.id]);
    },
    headerClick(setting) {
      if (!this.canExpand(setting)) return this.setView(`${setting.id}-main`);
      if (this.isSectionActive(setting)) return this.expandSection(setting);
      this.setView(`${setting.id}-${this.visibleSections(setting)[0].id}`);
    },
    expandSection(setting) {
      this.expandedSections[setting.id] = !this.expandedSections[setting.id];
    },
    isSectionActive(setting) {
      return this.activeView.startsWith(`${setting.id}-`);
    },
    isSubSectionActive(setting) {
      return this.isSectionActive(setting) && this.activeView !== `${setting.id}-main`;
    },
    visibleSections(setting) {
      return (setting.sections || []).filter((section) => this.shouldShow(section));
    },
    canExpand(setting) {
      if (setting.id === "profile") return this.showAdvancedProfile;
      return this.visibleSections(setting).length > 0;
    },
    closeSettings() {
      router.go(-1);
    },
    shouldShow(setting) {
      const perm = setting?.permissions || {};
      // Check if all keys in setting.perm exist in state.user.perm and have truthy values
      return Object.keys(perm).every((key) => getObjectProperty(state.user.permissions, key));
    },
    active: (view) => state.activeSettingsView === view,
    setView(view) {
      mutations.closeHovers();
      mutations.closeTopPrompt();
      void router.push({ path: "/settings", hash: `#${view}` }, () => {});
    },
    settingLabel(setting) {
      switch (setting.id) {
        case "profile":
          return this.$t("general.profileSettings");
        case "shares":
          return this.$t("general.shareSettings");
        case "users":
          return this.$t("general.userManagement");
        default:
          return this.$t(setting.label);
      }
    },
  },
};
</script>
<style>
/* Settings navigation uses the same flat pill rows as the files sidebar. */
.active-settings {
  background: var(--accent-container) !important;
  color: var(--on-accent-container) !important;
  font-weight: 600;
}

.settings-icon {
  font-size: 20px;
  color: inherit;
}

.active-settings .settings-icon,
.settings-card:hover .material-symbols-outlined {
  font-variation-settings: 'FILL' 1;
}

.settings-card {
  display: flex;
  align-items: center;
  overflow: unset !important;
  padding: 0 16px;
  min-height: 40px;
}

#sidebar .card.settings-card,
#sidebar .card.settings-card-collapsible {
  margin: 0 0 2px;
  border-radius: var(--radius-full);
  background: transparent;
  box-shadow: none;
  font-size: 14px;
}

/* The group wrapper is a plain container; each row inside gets its own pill. */
#sidebar .card.settings-card-collapsible {
  border-radius: 0;
}

#sidebar .card.settings-card:hover {
  background: var(--state-hover);
}

.settings-item-content {
  display: flex;
  align-items: center;
  gap: 12px;
}
</style>

<style scoped>
.settings-card-collapsible {
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
  overflow: hidden;
}

.settings-card-collapsible--active {
  outline: 0;
}

.settings-card-collapsible-chevron {
  margin-left: auto;
  transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.settings-card-collapsible-chevron.rotated {
  transform: rotate(180deg);
}

.settings-card-sub-item {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows 0.25s cubic-bezier(0.4, 0, 0.2, 1);
}

.settings-card-collapsible--expanded {
  grid-template-rows: 1fr;
}

.settings-card-sub-item-inner {
  overflow: hidden;
}

.settings-card-collapsible-header,
.settings-card-collapsible-sub-item,
.settings-card-collapsible .active-settings {
  margin: 0 0 2px;
  border-radius: var(--radius-full);
}

.settings-card-collapsible-sub-item {
  min-height: 36px;
  padding: 0 16px 0 44px;
}

.settings-card-collapsible-header:hover,
.settings-card-collapsible-sub-item:hover {
  background: var(--state-hover);
}
</style>
