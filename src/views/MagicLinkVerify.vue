<template>
	<div id="magic-link-page" class="is-flex is-justify-content-center is-align-items-center">
		<div class="verify-panel is-shadow">
			<div class="has-text-centered mb-5">
				<h1 class="title is-4">✨ Magic Link</h1>
				<p class="subtitle is-6">Verifying your magic link...</p>
			</div>

			<!-- Loading State -->
			<div v-if="verifying" class="has-text-centered">
				<b-loading :is-full-page="false" v-model="verifying" :can-cancel="false"></b-loading>
				<p class="mt-5">Please wait...</p>
			</div>

			<!-- Manual Code Entry (if no token in URL) -->
			<div v-else-if="!hasToken && !verified">
				<b-notification type="is-info" :closable="false">
					Enter the 6-digit code from your email
				</b-notification>

				<b-field label="Email">
					<b-input v-model="email" type="email" placeholder="your@email.com"></b-input>
				</b-field>

				<b-field label="Code">
					<b-input v-model="code" placeholder="ABC123" maxlength="6"
							 @input="code = code.toUpperCase()"
							 @keyup.enter.native="verifyCode"></b-input>
				</b-field>

				<b-button class="mt-4" expanded type="is-primary"
						  @click="verifyCode"
						  :loading="verifying"
						  :disabled="!email || code.length !== 6 || verified">
					Verify Code
				</b-button>
			</div>

			<!-- Success Message -->
			<div v-if="verified" class="has-text-centered">
				<b-icon icon="check-circle" size="is-large" type="is-success"></b-icon>
				<p class="title is-5 mt-3">Success!</p>
				<p>Redirecting to your dashboard...</p>
			</div>

			<!-- Error Message -->
			<b-notification v-if="error" type="is-danger" :closable="false">
				{{ error }}
			</b-notification>

			<!-- Back to Login -->
			<div class="has-text-centered mt-5">
				<router-link to="/login" class="has-text-link">
					← Back to login
				</router-link>
			</div>
		</div>
	</div>
</template>

<script>
export default {
	name: "magic-link-verify",
	data() {
		return {
			token: '',
			code: '',
			email: '',
			hasToken: false,
			verifying: false,
			verified: false,
			error: '',
		}
	},
	mounted() {
		// Check if token is in URL
		this.token = this.$route.query.token || ''
		this.hasToken = !!this.token

		// If token exists, verify immediately
		if (this.hasToken) {
			this.verifyToken()
		}
	},
	methods: {
		async verifyToken() {
			try {
				this.verifying = true
				this.error = ''

				const response = await this.$api.users.verifyMagicLink(this.token)

				if (response.data.success === 200) {
					await this.handleLoginSuccess(response.data.data)
				}
			} catch (err) {
				this.error = err.response?.data?.message || 'Invalid or expired magic link'
				this.verifying = false
			}
		},

		async verifyCode() {
			try {
				this.verifying = true
				this.error = ''

				const response = await this.$api.users.verifyMagicCode(this.email, this.code)

				if (response.data.success === 200) {
					await this.handleLoginSuccess(response.data.data)
				}
			} catch (err) {
				this.error = err.response?.data?.message || 'Invalid or expired code'
				this.verifying = false
			}
		},

		async handleLoginSuccess(data) {
			// Store tokens and user data
			localStorage.setItem("access_token", data.token.access_token)
			localStorage.setItem("refresh_token", data.token.refresh_token)
			localStorage.setItem("expires_at", data.token.expires_at)
			localStorage.setItem("user", JSON.stringify(data.user))

			this.$store.commit("SET_USER", data.user)
			this.$store.commit("SET_ACCESS_TOKEN", data.token.access_token)
			this.$store.commit("SET_REFRESH_TOKEN", data.token.refresh_token)

			// Get version
			try {
				const versionRes = await this.$api.sys.getVersion()
				if (versionRes.data.success == 200) {
					localStorage.setItem("version", versionRes.data.data.current_version)
				}
			} catch (err) {
				// Ignore version error
			}

			// Show success message and keep loading state to prevent button clicks
			this.verified = true
			// Keep verifying=true to keep button disabled during redirect

			// Redirect to home after 1 second
			setTimeout(() => {
				this.$router.push("/")
			}, 1000)
		}
	}
}
</script>

<style lang="scss" scoped>
#magic-link-page {
	height: calc(100% - 5.5rem);
	position: relative;
	z-index: 500;

	.verify-panel {
		text-align: left;
		background: rgba(255, 255, 255, 0.46);
		backdrop-filter: blur(1rem);
		border-radius: 8px;
		padding: 2.5rem 3rem;
		width: 28rem;
		max-width: 90vw;

		.label {
			color: #363636;
		}

		.input {
			background: rgba(255, 255, 255, 0.8);
			border-color: #dbdbdb;
		}
	}
}

@media screen and (max-width: 480px) {
	.verify-panel {
		margin: 0 1rem;
		padding: 2rem !important;
	}
}
</style>
