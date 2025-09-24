<script>
import isNull from 'lodash/isNull'
import YAML from 'yaml'
import FileSaver from 'file-saver'
import events from '@/events/events'
import cTooltip from '@/components/basicComponents/tooltip/tooltip.vue'
import business_ShowNewAppTag from '@/mixins/app/Business_ShowNewAppTag'
import business_OpenThirdApp from '@/mixins/app/Business_OpenThirdApp'
import business_LinkApp from '@/mixins/app/Business_LinkApp'
import tipEditorModal from '@/components/Apps/TipEditorModal.vue'
import commonI18n, { ice_i18n } from '@/mixins/base/common-i18n'

export default {
  name: 'AppCard',
  components: {
    CTooltip: cTooltip,
  },
  mixins: [business_ShowNewAppTag, business_OpenThirdApp, business_LinkApp, commonI18n],
  inject: ['homeShowFiles', 'openAppStore'],
  props: {
    item: {
      type: Object,
    },
  },
  data() {
    return {
      hover: false,
      dropState: false,
      isUninstalling: false,
      isCloning: false,
      isCheckThenUpdate: false,
      isUpdating: false,
      isRestarting: false,
      isStarting: false,
      isRebuilding: false,
      // isStoping: false,
      // Public. Only changes the state of the card, not the state of the button.
      isSaving: false,
      isActiveTooltip: false,
      dropdownPosition: 'is-bottom-right',
      // Web UI readiness tracking
      isWebUIInitializing: false,
      webUICheckStartTime: null,
      healthCheckTimer: null,
      healthCheckBackoff: 2000, // Start with 2 seconds
      maxHealthCheckBackoff: 30000, // Max 30 seconds between checks
      webUIReady: false,
    }
  },

  computed: {
    tooltipLabel() {
      if (this.isContainerApp) {
        return this.$t('Import to CasaOS')
      }
      else if (this.item.app_type === 'system') {
        return this.$t('Open')
      }
      else if (this.isUpdating) {
        return this.$t('Updating')
      }
      else if (this.isUninstalling) {
        return this.$t('Uninstalling')
      }
      else if (this.isCloning) {
        return this.$t('Cloning')
      }
      else if (this.isRestarting) {
        return this.$t('Restarting')
      }
      else if (this.isStarting) {
        return this.$t('Starting')
      }
      else if (this.isRebuilding) {
        return this.$t('Rebuilding')
      }
      else if (this.isCheckThenUpdate) {
        return this.$t('CheckThenUpdate')
      }
      else if (this.isWebUIInitializing && this.item.status === 'running') {
        return this.getWebUIWaitingMessage()
      }
      else if (this.item.status === 'running') {
        return this.$t('Open')
      }
      else {
        return this.$t('launch-and-open')
      }
    },
    tooltipTriger() {
      return ['hover']
      // if (this.isContainerApp || this.item.app_type === "system" || this.item.status === 'running') {
      //  eturn ['hover'];
      // } else {
      //  return [];
      // }
    },
    isLoading() {
      const active = this.isUninstalling || this.isUpdating || this.isRestarting || this.isStarting || this.isSaving || this.isRebuilding || this.isWebUIInitializing // || this.isStoping || this.isSaving
      return active
    },
    isV1App() {
      return this.item.app_type === 'v1app'
    },
    isV2App() {
      return this.item.app_type === 'v2app'
    },
    isContainerApp() {
      return this.item.app_type === 'container'
    },
    isLinkApp() {
      return this.item.app_type === 'LinkApp'
    },
    shutDownClass() {
      return this.item.status !== 'running' ? 'shutdown-rounded' : ''
    },
    hasWebUI() {
      // Skip system apps and LinkApps - they handle their own opening
      if (this.item.app_type === 'system' || this.item.app_type === 'LinkApp') {
        return false
      }

      // Apps with an 'index' property have web UI (from x-casaos configuration)
      return !!(this.item.index)
    },

  },

  watch: {
    'item.status'(newStatus, oldStatus) {
      if (newStatus === 'running' && oldStatus !== 'running' && this.hasWebUI) {
        // App just started and has web UI, begin health check
        this.webUIReady = false
        this.startWebUIHealthCheck()
      } else if (newStatus !== 'running') {
        // App stopped, clear health check
        this.stopWebUIHealthCheck()
        this.webUIReady = false
      }
    },
    hover(val) {
      if (!val && this.dropState)
        this.$refs.dro.toggle()
    },
    isLoading(active) {
      // design :: The first display is three seconds long
      if (this.isCheckThenUpdate && this.activeTimer === undefined) {
        this.activeTimer = setTimeout(() => {
          this.isActiveTooltip = false
          clearTimeout(this.activeTimer)
          this.activeTimer = undefined
        }, 3000)
        this.isActiveTooltip = true
      }
      else if (active === false && this.isCheckThenUpdate === false && this.activeTimer) {
        clearInterval(this.activeTimer)
        this.activeTimer = undefined
        this.isActiveTooltip = false
      }
    },
  },

  mounted() {
    // Check if app is already running when component loads and has web UI
    if (this.item.status === 'running' && this.item.app_type !== 'system' && !this.isLinkApp && this.hasWebUI) {
      this.startWebUIHealthCheck()
    }
  },

  beforeDestroy() {
    this.stopWebUIHealthCheck()
  },

  methods: {
    getWebUIWaitingMessage() {
      if (!this.webUICheckStartTime) return "Waiting for app interface..."

      const waitTime = Date.now() - this.webUICheckStartTime
      const minutes = Math.floor(waitTime / 60000)

      if (minutes < 0.5) return "Waiting for app interface..."
      if (minutes < 1) return "App is initializing..."
      if (minutes < 2) return "Still waiting for app to be ready..."
      if (minutes < 3) return "App is taking longer than usual..."
      return "App is still initializing, this may take several minutes..."
    },

    getIconCursorStyle() {
      // Show default cursor when web UI is initializing (not clickable but not forbidden)
      if (this.isWebUIInitializing && this.hasWebUI && this.item.status === 'running') {
        return { cursor: 'default' }
      }
      // Show pointer for clickable state
      return { cursor: 'pointer' }
    },

    async checkWebUIHealth() {
      if (this.item.app_type === 'system' || this.item.app_type === 'LinkApp') {
        this.webUIReady = true
        return true
      }

      try {
        if (this.isV2App) {
          const res = await this.$openAPI.appManagement.compose.checkComposeAppHealthByID(this.item.name)
          if (res.status === 200) {
            this.webUIReady = true
            return true
          }
        } else if (this.isV1App) {
          const res = await this.$api.container.containerLauncherCheck(this.item.name)
          if (res.data.success === 200) {
            this.webUIReady = true
            return true
          }
        }
        return false
      } catch (error) {
        return false
      }
    },

    startWebUIHealthCheck() {
      // Prevent multiple health check cycles
      if (this.isWebUIInitializing) {
        return
      }

      if (this.healthCheckTimer) {
        clearTimeout(this.healthCheckTimer)
      }

      this.isWebUIInitializing = true
      this.webUICheckStartTime = Date.now()
      this.webUIReady = false
      this.healthCheckBackoff = 2000

      this.performHealthCheckCycle()
    },

    async performHealthCheckCycle() {
      if (this.item.status !== 'running') {
        this.stopWebUIHealthCheck()
        return
      }

      const isHealthy = await this.checkWebUIHealth()

      if (isHealthy) {
        this.stopWebUIHealthCheck()
        return
      }

      this.healthCheckBackoff = Math.min(this.healthCheckBackoff * 1.5, this.maxHealthCheckBackoff)

      this.healthCheckTimer = setTimeout(() => {
        this.performHealthCheckCycle()
      }, this.healthCheckBackoff)
    },

    stopWebUIHealthCheck() {
      if (this.healthCheckTimer) {
        clearTimeout(this.healthCheckTimer)
        this.healthCheckTimer = null
      }
      this.isWebUIInitializing = false
      this.webUICheckStartTime = null
      this.healthCheckBackoff = 2000
    },

    handleDorpdownPosition(event) {
      this.$nextTick(() => {
        const rightOffset = window.innerWidth - event.clientX - 160
        const horizontalPos = rightOffset > 0 ? 'right' : 'left'
        const bottomOffset = window.innerHeight - event.clientY - 212
        const verticalPos = bottomOffset > 0 ? 'bottom' : 'top'
        this.dropdownPosition = `is-${verticalPos}-${horizontalPos}`
      })
    },
    /**
     * @description: Open app in new windows
     * @param {string} status App status
     * @param {string} port App access port
     * @param {string} index App access index
     * @return {*} void
     */
    openApp(item) {
      // Block clicks if web UI is initializing (patient waiting)
      if (this.isWebUIInitializing && this.hasWebUI && item.status === 'running') {
        return  // Do nothing - wait for web UI to be ready
      }

      if (this.isContainerApp) {
        this.$emit('importApp', item, false)
        return false
      }
      if (item.app_type === 'system') {
        this.openSystemApps(item)
      }
      else if (this.isLinkApp) {
        window.open(item.hostname, '_blank')
        this.removeIdFromSessionStorage(item.name)
      }
      else {
        // type is one of 'official' or 'community'.
        this.$refs.dro.isActive = false
        if (item.status === 'running') {
          if (this.webUIReady || !this.hasWebUI) {
            // App is ready OR has no web UI - open directly
            this.openAppToNewWindow(item)
          } else if (this.isWebUIInitializing) {
            // User clicked during initialization, send to loading page as fallback
            this.firstOpenThirdApp(item)
          } else if (this.hasWebUI) {
            // App has web UI but not ready yet - send to loading page
            this.firstOpenThirdApp(item)
          }
        }
        else {
          this.toggle(item)
          this.firstOpenThirdApp(item)
        }
      }
    },

    openSystemApps(item) {
      switch (item.name) {
        case 'App Store':
          this.openAppStore()
          break
        case 'Files':
          this.homeShowFiles()
          break
        default:
          break
      }
    },

    /**
     * @description: Set drop-down menu status
     * @param {boolean} e
     * @return {*} void
     */
    setDropState(e) {
      this.dropState = e
    },

    /**
     * @description: Restart Application
     * @return {*} void
     */
    restartApp() {
      this.$messageBus('apps_restart', this.item.name)
      this.isRestarting = true
      if (this.isV2App) {
        this.restartAppV2()
      }
      else if (this.isV1App) {
        this.restartAppV1()
      }
      this.$refs.dro.isActive = false
    },

    restartAppV1() {
      this.$api.container.updateState(this.item.name, 'restart').then((res) => {
        if (res.data.success === 200) {
          this.updateState()
        }
      }).catch((err) => {
        this.$buefy.toast.open({
          message: err.response.data.data || err.response.data.message,
          type: 'is-danger',
          position: 'is-top',
          duration: 5000,
        })
      }).finally(() => {
        this.isRestarting = false
      })
    },

    restartAppV2() {
      this.$openAPI.appManagement.compose.setComposeAppStatus(this.item.name, 'restart').then(() => {
        this.updateState()
      }).catch((err) => {
        this.$buefy.toast.open({
          message: err.response.data.data || err.response.data.message,
          type: 'is-danger',
          position: 'is-top',
          duration: 5000,
        })
      })
    },

    /**
     * @description: Confirm before uninstall
     * @return {*} void
     */
    uninstallConfirm() {
      this.$messageBus('apps_uninstall', this.item.name)
      this.$refs.dro.isActive = false
      this.$buefy.dialog.confirm({
        title: this.$t('Attention'),
        message: this.$t(`Data cannot be recovered after deletion! <br/>Continue on to uninstall this application?<br/>{divS}Delete userdata ( config folder ){divE}`, {
          divS: `<div class="is-flex is-align-items-center mt-4"><input type="checkbox"  id="checkDelConfig">`,
          divE: `</input></div>`,
        }),
        type: 'is-dark',
        confirmText: this.$t('Uninstall'),
        cancelText: this.$t('Cancel'),
        onConfirm: () => {
          const checkDelConfig = document.getElementById('checkDelConfig') ? document.getElementById('checkDelConfig').checked : false
          this.uninstallApp(checkDelConfig)
        },
      })
    },

    /**
     * @description: Uninstall app
     * @return {*} void
     */
    uninstallApp(checkDelConfig) {
      this.isUninstalling = true
      this.removeIdFromSessionStorage(this.item.name)
      if (this.isLinkApp) {
        this.deleteLinkAppByName(this.item.name).then((res) => {
          if (res.data.success === 200) {
            this.$EventBus.$emit(events.RELOAD_APP_LIST)
          }
        })
      }
      else if (this.isV2App) {
        this.$openAPI.appManagement.compose.uninstallComposeApp(this.item.name, checkDelConfig).then((res) => {
          if (res.status === 200) {
            this.$EventBus.$emit(events.UPDATE_SYNC_STATUS)
          }
        }).catch((err) => {
          this.$buefy.toast.open({
            message: err.response.data.data,
            type: 'is-danger',
            position: 'is-top',
            duration: 5000,
          })
        })
      }
      else {
        // former app uninstall
        this.$api.container.uninstall(this.item.name, { delete_config_folder: checkDelConfig }).then((res) => {
          if (res.data.success === 200) {
            this.$EventBus.$emit(events.UPDATE_SYNC_STATUS)
          }
        }).catch((err) => {
          this.$buefy.toast.open({
            message: err.response.data.data,
            type: 'is-danger',
            position: 'is-top',
            duration: 5000,
          })
        })
      }
    },

    /**
     * @description: Emit the event that the app has been updated
     * @return {*} void
     */
    updateState() {
      this.$refs.dro.isActive = false
      this.$emit('updateState')
      this.$EventBus.$emit(events.UPDATE_SYNC_STATUS)
    },

    async openTips(name) {
      try {
        const ret = await this.$openAPI.appManagement.compose.myComposeApp(name, {
          headers: {
            'content-type': 'application/yaml',
            'accept': 'application/yaml',
          },
        }).then(res => res.data)
        this.$refs.dro.isActive = false
        this.$buefy.modal.open({
          parent: this,
          component: tipEditorModal,
          hasModalCard: true,
          customClass: 'network-storage-modal',
          trapFocus: true,
          canCancel: [],
          // scroll: "keep",
          animation: 'zoom-in',
          props: {
            composeData: YAML.parse(ret),
            name,
          },
        })
      }
      catch (e) {
        console.error('openTips Error:', e)
      }
    },

    /**
     * @description: Emit the event that the app has been updated with custom_id
     * @return {*} void
     */
    configApp() {
      this.$messageBus('apps_setting', this.item.name)
      this.$refs.dro.isActive = false
      this.$emit('configApp', this.item, this.isV2App)
    },

    /**
     * @description: Start or Stop App
     * @param {object} item the app info object
     * @return {*} void
     */
    toggle(item) {
      // only have 'apps_stop' event
      this.$messageBus('apps_stop', item.name)
      this.isStarting = true
      const status = item.status === 'running' ? 'stop' : 'start'
      if (this.isV2App) {
        this.toggleAppV2(item, status)
      }
      else if (this.isV1App) {
        this.toggleAppV1(item, status)
      }
      this.$refs.dro.isActive = false
    },

    toggleAppV1(item, status) {
      this.$api.container.updateState(item.name, status).then((res) => {
        if (res.data.success === 200) {
          item.status = res.data.data
          this.updateState()
        }
        else {
          this.$buefy.dialog.alert({
            title: 'Error',
            message: res.data.data || res.data.message,
            type: 'is-danger',
            ariaRole: 'alertdialog',
            ariaModal: true,
          })
        }
      }).catch((err) => {
        this.$buefy.toast.open({
          message: err.response.data.data || err.response.data.message,
          type: 'is-danger',
          position: 'is-top',
          duration: 3000,
        })
      }).finally(() => {
        this.isStarting = false
      })
    },

    toggleAppV2(item, status) {
      this.$openAPI.appManagement.compose.setComposeAppStatus(item.name, status).then(() => {
        this.updateState()
        item.status = status
      }).catch((err) => {
        this.$buefy.dialog.alert({
          title: 'Error',
          message: err.response.data.data || err.response.data.message,
          type: 'is-danger',
          ariaRole: 'alertdialog',
          ariaModal: true,
        })
      })
    },

    appClone(name) {
      this.isCloning = true
      this.$api.apps.getAppInfo(name).then((resp) => {
        if (resp.data.success === 200) {
          const respData = resp.data.data
          // messageBus :: apps_clone
          this.$messageBus('apps_clone', this.item.name.toString())

          const initData = {}
          initData.protocol = respData.protocol
          initData.host = respData.host
          initData.port_map = respData.port_map
          initData.cpu_shares = 50
          initData.memory = respData.max_memory
          initData.restart = 'always'
          initData.label = respData.title
          initData.position = true
          initData.index = respData.index
          initData.icon = respData.icon
          initData.network_model = respData.network_model
          initData.image = respData.image
          initData.description = respData.description
          initData.origin = respData.origin
          initData.ports = isNull(respData.ports) ? [] : respData.ports
          initData.volumes = isNull(respData.volumes) ? [] : respData.volumes
          initData.envs = isNull(respData.envs) ? [] : respData.envs
          initData.devices = isNull(respData.devices) ? [] : respData.devices
          initData.cap_add = isNull(respData.cap_add) ? [] : respData.cap_add
          initData.cmd = isNull(respData.cmd) ? [] : respData.cmd
          initData.privileged = respData.privileged
          initData.host_name = respData.host_name
          initData.appstore_id = name

          this.$api.container.install(initData).catch((err) => {
            this.$buefy.toast.open({
              message: err.response.data.message,
              type: 'is-warning',
            })
          }).then(() => {
            this.isCloning = false
            this.$refs.dro.isActive = false
          })
        }
      }).catch(() => {
        this.$buefy.toast.open({
          message: this.$t(`There was an error loading the data, please try again!`),
          type: 'is-danger',
        })
      })
    },

    exportYAML(item) {
      this.$api.container.exportAsCompose(item.name).then((res) => {
        const blob = new Blob([res.data], { type: '' })
        FileSaver.saveAs(blob, `${item.image}.yaml`)
      }).catch((err) => {
        this.$buefy.toast.open({
          message: err.response.data.message,
          type: 'is-warning',
        })
      })
    },

    async rebuild(app) {
      this.isRebuilding = true
      try {
        // 1. get yaml
        const file = await this.$api.container.exportAsCompose(app.name).then(res => res.data)
        // 2. archive
        await this.$api.container.archive(app.name)
        // 3.install compose
        await this.$openAPI.appManagement.compose.installComposeApp(file, { name: app.name })
      }
      catch (e) {
        this.isRebuilding = false
        console.error('rebuild Error:', e)
        this.$buefy.toast.open({
          message: this.$t(`Rebulid error`),
          type: 'is-danger',
        })
      }
      // 4.sockiet :: install-end :: change UI status.
      // this.isRebuilding = false;
      this.$refs.dro.isActive = false
    },

    checkAppVersion(name) {
      this.isCheckThenUpdate = true
      this.$openAPI.appManagement.compose.updateComposeApp(name).then((resp) => {
        // 200:
        if (resp.status === 200) {
          // messageBus :: apps_checkThenUpdate
          this.$messageBus('apps_checkupdate', this.item.name.toString())
          this.$buefy.toast.open({
            // value is `In the process of asynchronous updating.` or `compose app `app Name` is up to date`
            message: resp.data.message,
            type: 'is-success',
          })
        }
        else {
          this.$buefy.toast.open({
            message: this.$t(`No updates are currently available for the application.`),
            type: 'is-success',
          })
        }
      }).catch(() => {
        this.$buefy.toast.open({
          message: this.$t(`Unable to update at the moment!`),
          type: 'is-danger',
        })
      }).finally(() => {
        this.$refs.dro.isActive = false
        this.isCheckThenUpdate = false
      })
    },
    /**
     * @description: Format Dot Class
     * @param {string} status
     * @return {string}
     */
    dotClass(status, loadState) {
      // For updating
      if (loadState) {
        if (status === '0' || status === 'running') {
          return 'disabled start'
        }
        return 'disabled stop'
      }
      if (status === '0') {
        return 'start'
      }
      else {
        return status === 'running' ? 'start' : 'stop'
      }
    },

  },

  sockets: {
    'app:start-error': function (res) {
      // toast info.
      this.$buefy.toast.open({
        message: res.Properties.message,
        duration: 5000,
        type: 'is-danger',
      })
    },
    'app:start-end': function (res) {
      if (res.Properties['app:name'] === this.item.name) {
        this.isRestarting = false
        this.isStarting = false
        if (this.hasWebUI) {
          this.startWebUIHealthCheck()
        }
      }
    },
    'app:stop-error': function (res) {
      // toast info.
      this.$buefy.toast.open({
        message: res.Properties.message,
        duration: 5000,
        type: 'is-danger',
      })
    },
    'app:stop-end': function (res) {
      if (res.Properties['app:name'] === this.item.name) {
        this.isRestarting = false
        this.isStarting = false
        this.stopWebUIHealthCheck()
        this.webUIReady = false
      }
    },
    'app:restart-error': function (res) {
      // toast info.
      this.$buefy.toast.open({
        message: res.Properties.message,
        duration: 5000,
        type: 'is-danger',
      })
    },
    'app:restart-end': function (res) {
      if (res.Properties['app:name'] === this.item.name) {
        this.isRestarting = false
        this.isStarting = false
        this.webUIReady = false
        if (this.hasWebUI) {
          this.startWebUIHealthCheck()
        }
      }
    },
    'app:apply-changes-begin': function (res) {
      if (res.Properties['app:name'] === this.item.name) {
        this.isSaving = true
      }
    },
    'app:apply-changes-error': function (res) {
      // toast info.
      this.$buefy.toast.open({
        message: res.Properties.message,
        duration: 5000,
        type: 'is-danger',
      })
    },
    'app:apply-changes-end': function (res) {
      if (res.Properties['app:name'] === this.item.name) {
        this.isRestarting = false
        this.isStarting = false
        this.isSaving = false
      }
    },
    /**
     * @description: Update App Status
     * @param {object} data
     * @return {void}
     */
    'app:update-begin': function () {

    },

    'docker:image:pull-end': function (data) {
      if (data.Properties['app:name'] === this.item.name) {
        if (data.Properties['docker:image:updated'] === 'true') {
          this.isUpdating = true
        }
        this.isCheckThenUpdate = false
      }
    },

    'docker:image:pull-error': function (data) {
      if (data.Properties['app:name'] === this.item.name) {
        this.isCheckThenUpdate = false
      }
    },

    /**
     * @description: Update App Version
     * @param {object} data
     * @return {void}
     */
    'app:update-end': function (data) {
      if (data.Properties['app:name'] !== this.item.name)
        return
      if (data.Properties['docker:image:updated'] === 'true') {
        return
      }
      this.isUpdating = false
      this.$buefy.toast.open({
        message: this.$t(`{appName} is the latest version!`, { appName: this.item.name }),
        type: 'is-success',
        duration: 5000,
      })
    },
    'app:install-end': function (res) {
      if (res.Properties['dry_run.name'] === this.item.name) {
        // 4.sockiet :: install-end :: change UI status.
        this.isRebuilding = false
        // 5.message toast
        this.$buefy.toast.open({
          message: this.$t(`{title} rebulid completed`, { title: ice_i18n(this.item.title) }),
          type: 'is-success',
        })
      }
    },
    'app:install-error': function (res) {
      if (res.Properties['dry_run.name'] === this.item.name) {
        // 4.sockiet :: install-end :: change UI status.
        this.isRebuilding = false
        // 5.message toast
        this.$buefy.toast.open({
          message: res.Properties.message,
          type: 'is-warning',
        })
      }
    },
    'app:uninstall-error': function (res) {
      if (res.Properties.id === this.item.name) {
        this.isUninstalling = false
      }
    },
  },

}
</script>

<template>
  <div
    :class="['common-card', 'is-flex', 'is-align-items-center', 'is-justify-content-center', 'app-card', { 'web-ui-initializing': isWebUIInitializing && hasWebUI && item.status === 'running' }]"
    @mouseleave="hover = true" @mouseover="hover = true"
  >
    <!-- Action Button Start -->
    <div v-if="item.app_type !== 'system' && !isContainerApp && !isUninstalling" class="action-btn">
      <b-dropdown
        ref="dro" :mobile-modal="false" :triggers="['contextmenu', 'click']" animation="fade1"
        append-to-body aria-role="list" class="app-card-drop" :position="dropdownPosition"
        @active-change="setDropState"
      >
        <template #trigger>
          <p role="button" @click="handleDorpdownPosition">
            <b-icon class="is-clickable" icon="dots-vertical-outline" pack="casa" />
          </p>
        </template>

        <b-dropdown-item :focusable="false" aria-role="menu-item" custom>
          <b-button v-if="item.status === 'running'" expanded tag="a" type="is-text" @click="openApp(item)">
            {{
              $t('Open') }}
          </b-button>
          <b-button v-else expanded tag="a" type="is-text" @click="openApp(item)">
            {{
              $t('launch-and-open') }}
          </b-button>
          <b-button
            v-if="isV2App" expanded icon-pack="casa" icon-right="question-outline" size="is-16"
            type="is-text" @click="openTips(item.name)"
          >
            {{ $t('Tips') }}
          </b-button>
          <b-button v-if="isV2App || isLinkApp" expanded type="is-text" @click="configApp()">
            {{
              $t('Setting')
            }}
          </b-button>

          <b-button v-if="isV2App && !item.is_uncontrolled" expanded type="is-text" @click="checkAppVersion(item.name)">
            {{
              $t('Check then update')
            }}
            <b-loading :active="isCheckThenUpdate || isUpdating" :is-full-page="false">
              <img :src="require('@/assets/img/loading/waiting.svg')" alt="pending" class="ml-4 is-24x24">
            </b-loading>
          </b-button>

          <b-button v-if="isV1App" expanded type="is-text" @click="exportYAML(item)">
            {{
              $t('Export as Compose')
            }}
          </b-button>

          <b-button v-if="isV1App" :loading="isRebuilding" expanded type="is-text" @click="rebuild(item)">
            {{
              $t('Rebuild')
            }}
          </b-button>

          <b-button v-if="isLinkApp" class="mb-1" expanded type="is-text" @click="uninstallApp(true)">
            {{ $t('Delete') }}
            <b-loading v-model="isUninstalling" :is-full-page="false">
              <img :src="require('@/assets/img/loading/waiting.svg')" alt="pending" class="ml-4 is-24x24">
            </b-loading>
          </b-button>
          <b-button v-else class="has-text-red" expanded type="is-text" @click="uninstallConfirm">
            {{ $t('Uninstall') }}
            <b-loading v-model="isUninstalling" :is-full-page="false">
              <img :src="require('@/assets/img/loading/waiting.svg')" alt="pending" class="ml-4 is-24x24">
            </b-loading>
          </b-button>

          <div v-if="!isLinkApp" class="gap">
            <div class="columns is-gapless _b-bor is-flex">
              <div class="column is-flex is-justify-content-center is-align-items-center">
                <b-button
                  :loading="isRestarting" expanded type="is-text" :disabled="item.status !== 'running'"
                  @click="restartApp"
                >
                  <b-icon custom-size="is-size-20px" icon="restart-outline" pack="casa" />
                </b-button>
              </div>
              <div class="column is-flex is-justify-content-center is-align-items-center">
                <b-button
                  :class="item.status" :loading="isStarting" class="has-text-red" expanded
                  type="is-text" @click="toggle(item)"
                >
                  <b-icon
                    custom-size="is-size-20px" icon="shutdown-outline" pack="casa"
                    :custom-class="shutDownClass"
                  />
                </b-button>
              </div>
            </div>
          </div>
        </b-dropdown-item>
      </b-dropdown>
    </div>
    <!-- Action Button End -->
    <div class="blur-background" />
    <div class="cards-content">
      <!-- Card Content Start -->
      <b-tooltip
        :always="isActiveTooltip" :animated="true" :label="tooltipLabel" :triggers="tooltipTriger"
        animation="fade1" class="in-card" type="is-white"
      >
        <div class="has-text-centered is-flex is-justify-content-center is-flex-direction-column pt-5 pb-3px img-c">
          <div class="is-flex is-justify-content-center">
            <div class="is-relative">
              <b-image
                :class="dotClass(item.status, isLoading)" :src="item.icon"
                :src-fallback="require('@/assets/img/app/default.svg')" class="is-64x64"
                :style="getIconCursorStyle()"
                webp-fallback=".jpg" @click.native="openApp(item)"
              />
              <!-- Unstable -->
              <CTooltip v-if="newAppIds.includes(item.name)" class="__position" content="NEW" />

              <!-- Web UI Initialization Animation Start -->
              <div
                v-if="isWebUIInitializing && !isUninstalling && !isUpdating && !isRestarting && !isStarting && !isSaving && !isRebuilding && item.status === 'running' && hasWebUI"
                class="web-ui-loading-overlay"
              >
                <div class="pulsing-ring"></div>
              </div>
              <!-- Web UI Initialization Animation End -->
            </div>

            <!-- Loading Bar Start -->
            <b-loading
              :active="isLoading" :can-cancel="false" :is-full-page="false"
              class="has-background-gray-800 op80 is-64x64"
              style="top: auto;bottom: auto; right: auto; left: auto; border-radius: 11.5px"
            >
              <img :src="require('@/assets/img/loading/waiting-white.svg')" alt="loading" class="is-20x20">
            </b-loading>
            <!-- Loading Bar End -->
          </div>

          <p class="mt-3 one-line">
            <a class="one-line" style="cursor:default">
              {{ i18n(item.title) }}
            </a>
          </p>
        </div>
      </b-tooltip>
      <!-- Card Content End -->
    </div>
  </div>
</template>

<style lang="scss">
.pb-3px {
  padding-bottom: 3px;
}

.shutdown-rounded {
  border-radius: 50%;
  background-color: #000;
  color: #fff;
}

.web-ui-loading-overlay {
  position: absolute;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 64px;
  height: 64px;
  top: 0;
  left: 0;
  pointer-events: none;
  z-index: 25;
}

// Disable hover effects during web UI initialization
.common-card.web-ui-initializing {
  &:hover {
    box-shadow: none !important;
  }
}

.pulsing-ring {
  width: 64px;
  height: 64px;
  border: 3px solid rgba(59, 130, 246, 0.9);
  border-radius: 11.5px;
  animation: pulse-ring 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}

@keyframes pulse-ring {
  0% {
    border-color: rgba(59, 130, 246, 1);
    box-shadow: 0 0 0 0 rgba(59, 130, 246, 0.8), inset 0 0 20px rgba(59, 130, 246, 0.6);
  }
  50% {
    border-color: rgba(59, 130, 246, 0.7);
    box-shadow: 0 0 0 8px rgba(59, 130, 246, 0), inset 0 0 25px rgba(59, 130, 246, 0.3);
  }
  100% {
    border-color: rgba(59, 130, 246, 1);
    box-shadow: 0 0 0 0 rgba(59, 130, 246, 0), inset 0 0 20px rgba(59, 130, 246, 0.6);
  }
}

.app-card-drop {
  .dropdown-menu {
    min-width: 10rem;

    .dropdown-content {
      padding: 4px !important;
      background: none;
      background: hsla(0, 0%, 100%, 1);
      border-radius: 10px;

      .dropdown-item {
        padding: 0;

        &>* {
          margin: 1px 0;
        }
      }

      .button {
        padding-left: 1rem;
        padding-right: 1rem;
        border-radius: 5px;

        span {
          line-height: 1.25rem !important;
          height: 1.25rem !important;
        }

        span+span i {
          color: hsla(208, 16%, 42%, 1);
        }

        &.is-text {
          text-decoration: none;
          justify-content: flex-start;
          outline: none;
          transition: all 0.2s;
          border: none !important;
          height: 2rem;
          font-size: 0.875rem;
          color: hsla(208, 20%, 20%, 1);

          &.running {
            color: #779e2a !important;
          }

          &.exited {
            color: #ff1616 !important;
          }
        }

        &.has-text-red {
          &:hover {
            background: hsla(18, 98%, 94%, 1);
          }

          &:active {
            background: hsla(18, 100%, 80%, 1);
          }
        }

        &:focus {
          background: none;
          box-shadow: none;
          outline: none;
        }

        &:hover {
          background-color: hsla(208, 16%, 96%, 1);
        }

        &:active {
          /* Gary/200 */
          background-color: hsla(208, 16%, 94%, 1);
        }
      }

      .gap {
        margin-left: -4px;
        margin-right: -4px;
      }

      ._b-bor {
        border-top: hsla(208, 16%, 94%, 1) 1px solid;

        .is-text {
          text-decoration: none;
          justify-content: center !important;
        }

        .column {
          margin-bottom: -4px;

          .button {
            margin: 4px;
            height: 2rem;
          }
        }

        .column:first-child {
          border-right: hsla(208, 16%, 94%, 1) 1px solid;
        }
      }

      /*common*/
      .loading-overlay {
        &.is-active {
          background: hsla(208, 16%, 96%, 1) !important;
          justify-content: flex-start;
        }

        .loading-background {
          background: none;
        }
      }

      .is-24x24 {
        width: 1.5rem;
        height: 1.5rem;
      }

    }
  }
}

.in-card.b-tooltip {
  &.is-top .tooltip-content {
    bottom: auto;
    top: -15%;
  }

  .tooltip-content {
    box-shadow: none;
    padding: 0.375rem 0.75rem;
    border-radius: 0.5rem;
    font-family: $family-sans-serif;
    font-style: normal;
    line-height: 1.25rem;
    font-feature-settings: 'pnum' on, 'lnum' on;

    color: hsla(208, 20%, 20%, 1);

  }

}

.__position {
  position: absolute !important;
  top: -0.75rem !important;
  left: 3rem !important;
  z-index: 30;
}

// 0.4.4
.dropdown.is-right .dropdown-menu {
  top: 0;
  left: calc(100% + 6px);
}

.dropdown.is-left .dropdown-menu {
  top: 0;
  left: calc(-100% - 14px);
}
</style>

<style lang="scss">
.dialog {
  .modal-card-head {
    padding-left: 1.5rem;
    padding-top: 1.5rem;
    padding-bottom: 0.75rem;
    border: 1px solid hsla(208, 16%, 94%, 1);
  }

  .modal-card-body {
    padding: 1rem 1.5rem 1.5rem;

    #checkDelConfig {
      margin-right: 0.5rem;
      height: 1.25rem;
      width: 1.25rem;
    }

    border: 1px solid hsla(208, 16%, 94%, 1);
  }

  .modal-card-foot {
    padding-top: 0.75rem;
    padding-bottom: 1.5rem;
    padding-right: 1.5rem;

    font-size: 14px;
    font-weight: 400;
    line-height: 20px;
    letter-spacing: 0;
    text-align: left;

    .button {
      margin-right: 0;
    }

    .is-dark {
      margin-left: 1rem;
      background: hsla(208, 100%, 45%, 1);
    }
  }
}
</style>
