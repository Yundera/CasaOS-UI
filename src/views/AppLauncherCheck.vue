<!--
 * @LastEditors: Jerryk jerry@icewhale.org
 * @LastEditTime: 2023-01-30 15:06:13
 * @FilePath: /CasaOS-UI/src/views/AppLauncherCheck.vue
  * @Description:
  *
  * Copyright (c) 2022 by IceWhale, All Rights Reserved.
  -->
<template>
	<div v-if="isCheckFailed"
		 class="is-flex is-flex-direction-column is-align-items-center is-justify-content-center is-fullheight">
		<b-image :key="appDetailData.icon" :src="appDetailData.icon"
				 :src-fallback="require('@/assets/img/app/default.svg')"
				 class="is-64x64 icon-shadow" webp-fallback=".jpg"></b-image>
		<h2 class="has-text-emphasis-01 has-text-white mt-2">{{ appDetailData.name }}</h2>
		<h1 v-if="status === 'pending'" class="has-text-sub-03 has-text-white mt-6">{{ $t('App is starting, please wait...') }}
		</h1>
		<h1 v-else-if="counter >= 60 && counter < checkCounts" class="has-text-sub-03 has-text-white mt-6">{{ $t('Taking longer than expected') }}</h1>
		<h1 v-else class="has-text-sub-03 has-text-white mt-6">{{ $t('APP may not be available') }}</h1>
		<b-image v-if="status === 'pending'" :src="require('@/assets/img/loading/waiting.svg')" alt="pending"
				 class="is-48x48 mt-6"/>
		<span v-else class="has-text-full-03 has-text-grey-600 mt-6">{{
				$t('Please')
			}}
      <a @click="openThirdApp(appDetailData);">{{ $t('Click here') }}
      </a> {{ $t('to open the app. If it does not work, please restart or try again later.') }}
    </span>
		<img :src="require('@/assets/img/logo/logo.svg')" alt="" class="is-absolute position"/>
	</div>
</template>

<script>
import business_OpenThirdApp from "@/mixins/app/Business_OpenThirdApp";

export default {
	name: "AppLauncherCheck",
	mixins: [business_OpenThirdApp],
	data() {
		return {
			appDetailData: {
				icon: "",
				name: ""
			},
			status: "pending",
			timer: null,
			isCheckFailed: false,
			checkCounts: 1000,
			counter: 0,
			currentInterval: 2000
		}
	},

	async created() {
		this.appDetailData = JSON.parse(this.$route.query.appDetailData)
		const startRes = await this.startContainer()
		this.scheduleNextCheck()
		this.check()
	},

	methods: {
		// Get container running state
		async getContainerState() {
			try {
				let res = await this.$api.container.getState(this.appDetailData.name)
				return res.data.data
			} catch (error) {
				return {
					state: "error"
				}
			}
		},
		// Start container
		async startContainer() {
			try {
				let res = await this.$openAPI.appManagement.compose.setComposeAppStatus(this.appDetailData.name, "start")
				return res.data
			} catch (error) {
				return "error"
			}
		},
		// Check container health
		async healthCheck() {
			try {
				let res = await this.$openAPI.appManagement.compose.checkComposeAppHealthByID(this.appDetailData.name)
				return res.status === 200
			} catch (error) {
				return false
			}
		},

		calculateInterval() {
			const baseInterval = 2000
			const maxInterval = 10000
			const increaseEvery = 10

			const additionalTime = Math.floor(this.counter / increaseEvery) * 1000
			const interval = baseInterval + additionalTime

			return Math.min(interval, maxInterval)
		},

		scheduleNextCheck() {
			this.timer && clearTimeout(this.timer)
			this.currentInterval = this.calculateInterval()
			this.timer = setTimeout(this.check, this.currentInterval)
		},

		async check() {
			this.counter += 1
			const isOk = await this.healthCheck()
			if (isOk) {
				clearTimeout(this.timer)
				this.openThirdApp(this.appDetailData)
			} else if (this.counter >= this.checkCounts) {
				this.status = "reject"
				clearTimeout(this.timer)
			} else {
				this.isCheckFailed = true
				if (this.counter >= 60) {
					this.status = "fallback"
				}
				this.scheduleNextCheck()
			}
		}
	}
}
</script>

<style lang="scss" scoped>
.is-fullheight {
	background: hsla(208, 20%, 12%, 1);
	height: 100vh;
}

.position {
	left: 2rem;
	bottom: 1.25rem;
}
</style>