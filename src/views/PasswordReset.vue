<template>
	<div id="password-reset-page" class="is-flex is-justify-content-center is-align-items-center">
		<div class="reset-panel is-shadow">
			<div class="has-text-centered mb-5">
				<h1 class="title is-4">🔑 Reset Password</h1>
				<p class="subtitle is-6" v-if="!tokenVerified">Enter your reset code</p>
				<p class="subtitle is-6" v-else>Choose a new password</p>
			</div>

			<!-- Step 1: Verify Token/Code -->
			<div v-if="!tokenVerified">
				<!-- Loading State -->
				<div v-if="verifying" class="has-text-centered">
					<b-loading :is-full-page="false" v-model="verifying" :can-cancel="false"></b-loading>
					<p class="mt-5">Verifying...</p>
				</div>

				<!-- Manual Code Entry -->
				<div v-else>
					<b-notification type="is-info" :closable="false">
						Enter the 6-digit code from your email
					</b-notification>

					<b-field label="Email">
						<b-input v-model="email" type="email" placeholder="your@email.com"></b-input>
					</b-field>

					<b-field label="Code">
						<b-input v-model="code" placeholder="ABC123" maxlength="6"
								 @input="code = code.toUpperCase()"
								 @keyup.enter.native="verifyResetCode"></b-input>
					</b-field>

					<b-button class="mt-4" expanded type="is-primary"
							  @click="verifyResetCode"
							  :loading="verifying"
							  :disabled="!email || code.length !== 6 || tokenVerified">
						Verify Code
					</b-button>
				</div>
			</div>

			<!-- Step 2: Set New Password -->
			<div v-if="tokenVerified && !resetComplete">
				<b-notification type="is-success" :closable="false">
					Code verified! Now set your new password.
				</b-notification>

				<ValidationObserver ref="observer" v-slot="{ handleSubmit }">
					<ValidationProvider v-slot="{ errors, valid }" name="Password" rules="required|min:6" vid="password">
						<b-field label="New Password" :message="errors"
								 :type="{ 'is-danger': errors[0], 'is-success': valid }">
							<b-input v-model="newPassword" type="password" password-reveal></b-input>
						</b-field>
					</ValidationProvider>

					<ValidationProvider v-slot="{ errors, valid }" name="Password Confirmation" rules="required|confirmed:password">
						<b-field label="Confirm Password" :message="errors"
								 :type="{ 'is-danger': errors[0], 'is-success': valid }">
							<b-input v-model="newPasswordConfirm" type="password" password-reveal
									 @keyup.enter.native="handleSubmit(submitNewPassword)"></b-input>
						</b-field>
					</ValidationProvider>

					<b-button class="mt-4" expanded type="is-primary"
							  @click="handleSubmit(submitNewPassword)"
							  :loading="submitting"
							  :disabled="submitting || resetComplete">
						Reset Password
					</b-button>
				</ValidationObserver>
			</div>

			<!-- Success Message -->
			<div v-if="resetComplete" class="has-text-centered">
				<b-icon icon="check-circle" size="is-large" type="is-success"></b-icon>
				<p class="title is-5 mt-3">Password Reset Successfully!</p>
				<p>Redirecting to login...</p>
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
import {ValidationObserver, ValidationProvider} from "vee-validate"
import "@/plugins/vee-validate"

export default {
	name: "password-reset",
	components: {
		ValidationObserver,
		ValidationProvider,
	},
	data() {
		return {
			token: '',
			code: '',
			email: '',
			tokenId: null,
			hasToken: false,
			verifying: false,
			tokenVerified: false,
			newPassword: '',
			newPasswordConfirm: '',
			submitting: false,
			resetComplete: false,
			error: '',
		}
	},
	mounted() {
		// Check if token_id and email are passed (already verified from code)
		const tokenId = this.$route.query.token_id
		const email = this.$route.query.email

		if (tokenId && email) {
			// Code was already verified, skip to password reset
			this.tokenId = parseInt(tokenId)
			this.email = email
			this.tokenVerified = true
			return
		}

		// Check if token is in URL (from magic link)
		this.token = this.$route.query.token || ''
		this.hasToken = !!this.token

		// If token exists, verify immediately
		if (this.hasToken) {
			this.verifyResetToken()
		}
	},
	methods: {
		async verifyResetToken() {
			try {
				this.verifying = true
				this.error = ''

				const response = await this.$api.users.verifyPasswordResetToken(this.token)

				if (response.data.success === 200) {
					this.tokenId = response.data.data.token_id
					this.email = response.data.data.email
					this.tokenVerified = true
				}
			} catch (err) {
				this.error = err.response?.data?.message || 'Invalid or expired reset link'
			} finally {
				this.verifying = false
			}
		},

		async verifyResetCode() {
			try {
				this.verifying = true
				this.error = ''

				const response = await this.$api.users.verifyPasswordResetCode(this.email, this.code)

				if (response.data.success === 200) {
					this.tokenId = response.data.data.token_id
					this.tokenVerified = true
				}
			} catch (err) {
				this.error = err.response?.data?.message || 'Invalid or expired code'
			} finally {
				this.verifying = false
			}
		},

		async submitNewPassword() {
			try {
				this.submitting = true
				this.error = ''

				await this.$api.users.confirmPasswordReset(this.tokenId, this.email, this.newPassword)

				this.resetComplete = true

				// Redirect to login after 2 seconds
				setTimeout(() => {
					this.$router.push("/login")
				}, 2000)
			} catch (err) {
				this.error = err.response?.data?.message || 'Failed to reset password'
			} finally {
				this.submitting = false
			}
		}
	}
}
</script>

<style lang="scss" scoped>
#password-reset-page {
	height: calc(100% - 5.5rem);
	position: relative;
	z-index: 500;

	.reset-panel {
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
	.reset-panel {
		margin: 0 1rem;
		padding: 2rem !important;
	}
}
</style>
