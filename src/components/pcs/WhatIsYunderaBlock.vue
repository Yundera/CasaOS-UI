<template>
	<div class="common-card">
		<div class="blur-background"></div>
		<div class="wuji-content _box is-flex is-flex-direction-column">
			<!-- What is Yundera Block Start -->
			<h6 class="title is-4 mb-0 has-text-white is-flex-shrink-0">{{ $t(`What is Yundera?`) }}</h6>
			<div class="is-flex is-align-items-center is-flex-grow-1 _notice-content-text">
				<div class="info">
					<div class="des is-size-14px">
						{{ $t(`Explore open-source tools that replace everyday apps: like editing PDFs, music or movies all on your own private cloud.`) }}
					</div>
				</div>
				<div class="emoji-icon is-80x80">🧠</div>
			</div>
			<div class="buttons is-flex is-flex-shrink-0 is-flex-direction-row-reverse">
				<b-button 
					class="mb-0" 
					rounded 
					size="is-small" 
					type="is-primary" 
					@click="openWhatIsYundera"
					:loading="isLoading"
				>
					{{ $t(`What is a private cloud server?`) }}
				</b-button>
			</div>
			<!-- What is Yundera Block End -->
		</div>
	</div>
</template>

<script>
export default {
	name: "what-is-yundera-block",
	data() {
		return {
			isLoading: false
		}
	},
	methods: {
		openWhatIsYundera() {
			this.isLoading = true;

			window.open('https://www.yundera.com/blog/tutorial-1/howtostart-34', '_blank');

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