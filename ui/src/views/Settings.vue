<template>
  <div class="bx--grid bx--grid--full-width">
    <div class="bx--row">
      <div class="bx--col-lg-16 page-title">
        <h2>{{ $t("settings.title") }}</h2>
      </div>
    </div>
    <div v-if="error.getConfiguration" class="bx--row">
      <div class="bx--col">
        <NsInlineNotification
          kind="error"
          :title="$t('action.get-configuration')"
          :description="error.getConfiguration"
          :showCloseButton="false"
        />
      </div>
    </div>
    <div class="bx--row">
      <div class="bx--col-lg-16">
        <cv-tile :light="true">
          <cv-form @submit.prevent="saveSettings">
            <cv-text-input
              :label="$t('settings.phpmyadmin_path')"
              placeholder="/phpmyadmin"
              v-model.trim="path"
              class="mg-bottom"
              :invalid-message="$t(error.path)"
              :disabled="loading.settings"
              ref="path"
            >
            </cv-text-input>
            <template v-if="mariadb_tcp_port">
              <span class="mg-bottom"> 
                {{ $t('settings.mariadb_tcp_port')}}
                <cv-tooltip
                  alignment="start"
                  direction="bottom"
                  :tip="$t('settings.mariadb_tcp_port_tips')"
                  class="info mg-bottom"
                >
                </cv-tooltip>
              </span>
              <span>:</span>
              <span  class="mg-bottom mg-left" > 
                {{mariadb_tcp_port}} 
              </span>
            <section>
              <span> {{$t('settings.phpmyadmin_url')}}
                <cv-tooltip
                  alignment="start"
                  direction="bottom"
                  :tip="$t('settings.admin_login_tips')"
                  class="info mg-bottom"
                >
                </cv-tooltip>
              </span>
              <span>:</span>
              <cv-link class="mg-bottom mg-left"
                :href="'http://'+ hostname + path"
                target="_blank"
                :inline= false
                >
                {{$t('settings.link')}}
              </cv-link>
            </section>
            </template>
            <!-- <cv-toggle
              value="letsEncrypt"
              :label="$t('settings.lets_encrypt')"
              v-model="isLetsEncryptEnabled"
              :disabled="loading.settings"
              class="mg-bottom"
            >
              <template slot="text-left">{{
                $t("settings.disabled")
              }}</template>
              <template slot="text-right">{{
                $t("settings.enabled")
              }}</template>
            </cv-toggle> -->
            <cv-toggle
              value="httpToHttps"
              :label="$t('settings.http_to_https')"
              v-model="isHttpToHttpsEnabled"
              :disabled="loading.settings"
              class="mg-bottom"
            >
              <template slot="text-left">{{
                $t("settings.disabled")
              }}</template>
              <template slot="text-right">{{
                $t("settings.enabled")
              }}</template>
            </cv-toggle>
            <div v-if="error.configureModule" class="bx--row">
              <div class="bx--col">
                <NsInlineNotification
                  kind="error"
                  :title="$t('action.configure-module')"
                  :description="error.configureModule"
                  :showCloseButton="false"
                />
              </div>
            </div>
            <NsButton
              kind="primary"
              :icon="Save20"
              :loading="loading.settings"
              :disabled="loading.settings"
              >{{ $t("settings.save") }}</NsButton
            >
          </cv-form>
        </cv-tile>
      </div>
    </div>
  </div>
</template>

<script>
import to from "await-to-js";
import { mapState } from "vuex";
import {
  QueryParamService,
  UtilService,
  TaskService,
  IconService,
} from "@nethserver/ns8-ui-lib";

export default {
  name: "Settings",
  mixins: [TaskService, IconService, UtilService, QueryParamService],
  pageTitle() {
    return this.$t("settings.title") + " - " + this.appName;
  },
  data() {
    return {
      q: {
        page: "settings",
      },
      hostname: location.hostname,
      urlCheckInterval: null,
      path: "",
      mariadb_tcp_port: "",
      // isLetsEncryptEnabled: false,
      isHttpToHttpsEnabled: false,
      loading: {
        settings: true,
      },
      error: {
        getConfiguration: "",
        configureModule: "",
        path: "",
        lets_encrypt: "",
        http2https: "",
      },
    };
  },
  computed: {
    ...mapState(["instanceName", "core", "appName"]),
  },
  created() {
    this.getConfiguration();
  },
  beforeRouteEnter(to, from, next) {
    next((vm) => {
      vm.watchQueryData(vm);
      vm.urlCheckInterval = vm.initUrlBindingForApp(vm, vm.q.page);
    });
  },
  beforeRouteLeave(to, from, next) {
    clearInterval(this.urlCheckInterval);
    next();
  },
  methods: {
    async getConfiguration() {
      this.loading.settings = true;
      this.error.getConfiguration = "";
      const taskAction = "get-configuration";

      // register to task completion
      this.core.$root.$once(
        taskAction + "-completed",
        this.getConfigurationCompleted
      );

      const res = await to(
        this.createModuleTaskForApp(this.instanceName, {
          action: taskAction,
          extra: {
            title: this.$t("action." + taskAction),
            isNotificationHidden: true,
          },
        })
      );
      const err = res[0];

      if (err) {
        console.error(`error creating task ${taskAction}`, err);
        this.error.getConfiguration = this.getErrorMessage(err);
        return;
      }
    },
    validateSaveSettings() {
      this.clearErrors(this);
      let isValidationOk = true;
      if (!this.path) {
        this.error.path = "common.required";
        if (isValidationOk) {
          this.focusElement("path");
        }
        isValidationOk = false;
      }
      return isValidationOk;
    },
    saveSettingsValidationFailed(validationErrors) {
      this.loading.settings = false;
      let focusAlreadySet = false;

      for (const validationError of validationErrors) {
        const param = validationError.parameter;
        // set i18n error message
        this.error[param] = "settings." + validationError.error;

        if (!focusAlreadySet) {
          this.focusElement(param);
          focusAlreadySet = true;
        }
      }
    },
    async saveSettings() {
      const isValidationOk = this.validateSaveSettings();
      if (!isValidationOk) {
        return;
      }

      this.loading.settings = true;
      const taskAction = "configure-module";

      // register to task validation
      this.core.$root.$off(taskAction + "-validation-failed");
      this.core.$root.$once(
        taskAction + "-validation-failed",
        this.saveSettingsValidationFailed
      );

      // register to task completion
      this.core.$root.$off(taskAction + "-completed");
      this.core.$root.$once(
        taskAction + "-completed",
        this.saveSettingsCompleted
      );

      const res = await to(
        this.createModuleTaskForApp(this.instanceName, {
          action: taskAction,
          data: {
            path: this.path,
            // lets_encrypt: this.isLetsEncryptEnabled,
            http2https: this.isHttpToHttpsEnabled,
          },
          extra: {
            title: this.$t("settings.instance_configuration", {
              instance: this.instanceName,
            }),
            description: this.$t("settings.configuring"),
          },
        })
      );
      const err = res[0];

      if (err) {
        console.error(`error creating task ${taskAction}`, err);
        this.error.configureModule = this.getErrorMessage(err);
        this.loading.settings = false;
        return;
      }
    },
    getConfigurationCompleted(taskContext, taskResult) {
      const config = taskResult.output;
      this.path = config.path;
      this.mariadb_tcp_port = config.mariadb_tcp_port;
      // this.isLetsEncryptEnabled = config.lets_encrypt;
      this.isHttpToHttpsEnabled = config.http2https;
      this.loading.settings = false;
      this.focusElement("path");
    },
    saveSettingsCompleted() {
      this.loading.settings = false;

      // reload configuration
      this.getConfiguration();
    },
  },
};
</script>

<style scoped lang="scss">
@import "../styles/carbon-utils";
.mg-bottom {
  margin-bottom: $spacing-06;
}
.mg-left{
    margin-left: $spacing-05;
}
</style>
