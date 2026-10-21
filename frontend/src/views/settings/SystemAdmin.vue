<template>
  <div class="card-content">
    <div class="settings-items">
      <ToggleSwitch class="item" v-model="localuser.disableUpdateNotifications" @change="updateSettings"
        :name="$t('profileSettings.disableUpdateNotifications')"
        :description="$t('profileSettings.disableUpdateNotificationsDescription')" />
      <ToggleSwitch class="item" v-model="analyticsEnabled" :disabled="analyticsLoading || !publishSupported"
        @update:modelValue="updateAnalytics" :name="$t('settings.analyticsEnabled')"
        :description="$t('settings.analyticsEnabledDescription')" />
      <SettingsButton class="item" :name="$t('settings.analyticsView')"
        :description="$t('settings.analyticsViewDescription')" @click="openAnalyticsPrompt" />
      <SettingsButton class="item" :name="$t('settings.configViewerOpen')"
        :description="$t('settings.configViewerDescription')" @click="openConfigViewerPrompt" />
    </div>

    <!-- Branding: the name and icon come from config.yaml (frontend.name / frontend.loginIcon) and the
         server has no endpoint to change them at runtime, so the controls show the live values but stay disabled. -->
    <section class="branding" aria-disabled="true">
      <h3 class="branding__title">{{ $t("settings.brandingTitle") }}</h3>
      <p class="branding__hint">
        <i class="material-symbols-outlined" aria-hidden="true">lock</i>
        <span>{{ $t("settings.brandingManagedByConfig") }}</span>
      </p>
      <div class="branding__row">
        <label class="branding__label" for="branding-name">
          <span>{{ $t("settings.brandingName") }}</span>
          <small>{{ $t("settings.brandingNameDescription") }}</small>
        </label>
        <input id="branding-name" class="input branding__input" type="text" :value="brandName" disabled />
      </div>
      <div class="branding__row">
        <div class="branding__label">
          <span>{{ $t("settings.brandingIcon") }}</span>
          <small>{{ $t("settings.brandingIconDescription") }}</small>
        </div>
        <div class="branding__icon">
          <img v-if="brandIcon" :src="brandIcon" alt="" class="branding__preview" />
          <button type="button" class="button button--flat" disabled>
            <i class="material-symbols-outlined" aria-hidden="true">upload</i>
            <span>{{ $t("settings.brandingChangeIcon") }}</span>
          </button>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
import { notify } from "@/notify";
import { state, mutations } from "@/store";
import { globalVars } from "@/utils/constants";
import { getAnalytics, updateAnalytics } from "@/api/settings";
import ToggleSwitch from "@/components/settings/ToggleSwitch.vue";
import SettingsButton from "@/components/settings/SettingsButton.vue";

export default {
  name: "systemAdmin",
  components: {
    ToggleSwitch,
    SettingsButton,
  },
  data() {
    return {
      localuser: { disableUpdateNotifications: false },
      analyticsEnabled: false,
      publishSupported: false,
      analyticsLoading: false,
    };
  },
  computed: {
    brandName() {
      return globalVars.name || "";
    },
    brandIcon() {
      return globalVars.loginIcon || "";
    },
  },
  async mounted() {
    this.localuser = {
      disableUpdateNotifications: false,
      ...state.user,
    };
    await this.loadAnalytics();
  },
  methods: {
    async updateSettings(event) {
      if (event && typeof event.preventDefault === "function") {
        event.preventDefault();
      }
      try {
        void mutations.updateCurrentUser(this.localuser);
        notify.showSuccessToast(this.$t("settings.settingsUpdated"));
      } catch (e) {
        console.error(e);
      }
    },
    async loadAnalytics() {
      try {
        const status = await getAnalytics();
        this.analyticsEnabled = status.enabled;
        this.publishSupported = status.publishSupported;
      } catch (e) {
        console.error(e);
        notify.showErrorToast(e?.message || this.$t("settings.analyticsLoadFailed"));
      }
    },
    async updateAnalytics() {
      if (!this.publishSupported) {
        this.analyticsEnabled = false;
        return;
      }
      this.analyticsLoading = true;
      try {
        const status = await updateAnalytics({ enabled: this.analyticsEnabled });
        this.analyticsEnabled = status.enabled;
        notify.showSuccessToast(this.$t("settings.settingsUpdated"));
      } catch (e) {
        console.error(e);
        notify.showErrorToast(e?.message || this.$t("settings.analyticsUpdateFailed"));
        await this.loadAnalytics();
      } finally {
        this.analyticsLoading = false;
      }
    },
    openAnalyticsPrompt() {
      mutations.showPrompt({
        name: "analytics-diagnostic",
        props: {
          title: this.$t("settings.analyticsView"),
        },
      });
    },
    openConfigViewerPrompt() {
      mutations.showPrompt({
        name: "config-viewer",
        props: {
          title: this.$t("settings.configViewer"),
        },
      });
    },
  },
};
</script>

<style scoped>
.card-content {
  margin-top: 1em;
}

.branding {
  margin-top: 24px;
  padding: 20px 16px 8px;
  border-top: 1px solid var(--divider);
}

.branding__title {
  margin: 0 0 8px;
  font-family: var(--font-display);
  font-size: 16px;
  font-weight: 500;
  text-align: left;
}

.branding__hint {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  margin: 0 0 16px;
  padding: 10px 12px;
  border-radius: var(--radius-s);
  background: var(--alt-background);
  color: var(--textSecondary);
  font-size: 13px;
}

.branding__hint .material-symbols-outlined {
  font-size: 18px;
}

.branding__row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 24px;
  padding: 12px 0;
  opacity: 0.6;
}

.branding__label {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}

.branding__label small {
  color: var(--textSecondary);
  font-size: 12px;
}

.branding__input {
  max-width: 320px;
  cursor: not-allowed;
}

.branding__icon {
  display: flex;
  align-items: center;
  gap: 12px;
}

.branding__preview {
  width: 40px;
  height: 40px;
  object-fit: contain;
  border-radius: var(--radius-s);
  background: var(--alt-background);
}
</style>
