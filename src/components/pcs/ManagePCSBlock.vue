<template>
	<div class="common-card">
		<div class="blur-background"></div>
		<div class="wuji-content _box is-flex is-flex-direction-column">
			<!-- Manage PCS & Billing Block Start -->
			<h6 class="title is-4 mb-0 has-text-white is-flex-shrink-0">{{ $t(`Manage Your Private Cloud Server`) }}</h6>
			<div class="is-flex is-align-items-center is-flex-grow-1 _notice-content-text">
				<div class="info">
					<div class="des is-size-14px">
						{{ $t(`Restart your server, update payments, or change your domain.`) }}
					</div>
				</div>
				<div class="emoji-icon is-80x80">🛠️</div>
			</div>
			<div class="buttons is-flex is-flex-shrink-0 is-flex-direction-row-reverse">
				<b-button 
					class="mb-0" 
					rounded 
					size="is-small" 
					type="is-primary" 
					@click="openYunderaAdminPanel"
					:loading="isLoading"
				>
					{{ $t(`PCS Settings`) }}
				</b-button>
			</div>
			<!-- Manage PCS & Billing Block End -->
		</div>
	</div>
</template>

<script>
export default {
	name: "manage-pcs-block",
	data() {
		return {
			isLoading: false
		}
	},
	methods: {
		openYunderaAdminPanel() {
			this.isLoading = true;

			window.open('https://www.yundera.com/blog/tutorial-1/the-most-famous-open-source-apps-35', '_blank');

			// Reset loading state after a short delay
			setTimeout(() => {
				this.isLoading = false;
			}, 1000);
		}
	}
}
</script>

<style lang="scss" scoped>
._box {
	height: auto;
	min-height: 10rem;
	padding: 1rem 1.25rem;
	padding-top: 1.125rem;
	margin-bottom: 1rem;

	._notice-content-text {
		margin-right: 2rem;
		margin-left: 2rem;
	}
}

.des {
	line-height: 1.6em;
}

.emoji-icon {
	width: 80px;
	height: 80px;
	display: flex;
	align-items: center;
	justify-content: center;
	font-size: 48px;
	flex-shrink: 0;
}
</style>