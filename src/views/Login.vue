<template>
	<div id="login-page" class="is-flex is-justify-content-center is-align-items-center ">
		<div v-if="!isLoading" class="login-panel step4 is-shadow">
			<div class="is-flex is-justify-content-center pb-3 ">
				<div class="has-text-centered">
					<b-image :src-fallback="require('@/assets/img/account/default-avatar.svg')" src="/v1/users/image?path=/var/lib/casaos/1/avatar.png" class="is-128x128" rounded></b-image>
				</div>

			</div>
			<b-notification v-model="notificationShow" aria-close-label="Close notification" auto-close role="alert"
							:type="notificationType">
				{{ message }}
			</b-notification>

			<div class="mode-container">
				<transition name="fade-transform">
					<!-- Normal Login Mode -->
					<div v-if="mode === 'normal'" key="normal">
					<ValidationObserver ref="observer" v-slot="{ handleSubmit }">
				<ValidationProvider v-slot="{ errors, valid }" name="User" rules="required">
					<b-field :label="$t('Username')" :message="errors"
							 :type="{ 'is-danger': errors[0], 'is-success': valid }"
							 class="mt-3">
						<b-input v-model="username" :autofocus="!username" type="text" v-on:keyup.enter.native="handleSubmit(login)"></b-input>
					</b-field>
				</ValidationProvider>
				<ValidationProvider v-slot="{ errors, valid }" name="Password" rules="required|min:5" vid="password">
					<b-field :label="$t('Password')" :message="$t(errors)"
							 :type="{ 'is-danger': errors[0], 'is-success': valid }" class="mt-2">
						<b-input v-model="password" :autofocus="username" password-reveal
								 type="password" v-on:keyup.enter.native="handleSubmit(login)"></b-input>
					</b-field>
				</ValidationProvider>

				<!-- Forgot Password Link -->
				<div class="has-text-centered mt-3">
					<a @click="mode = 'password-reset'" class="has-text-link is-size-7" style="cursor: pointer;">
						{{ $t('Forgot password?') }}
					</a>
				</div>

				<b-button class="mt-4" expanded rounded type="is-primary" @click="handleSubmit(login)">{{ $t('Login') }}
				</b-button>

				<!-- Magic Link Button -->
				<div class="has-text-centered mt-4">
					<a @click="mode = 'magic-link'" class="has-text-link is-size-7" style="cursor: pointer; opacity: 0.8;">
						{{ $t('Sign in with magic link') }}
					</a>
				</div>
					</ValidationObserver>
				</div>

				<!-- Magic Link / Password Reset Mode -->
				<div v-else-if="mode === 'magic-link' || mode === 'password-reset'" :key="mode">
				<h3 class="title is-5 has-text-centered mb-4">
					{{ mode === 'magic-link' ? $t('Sign in with magic link') : $t('Reset your password') }}
				</h3>

				<!-- Email Input (show when email not sent yet) -->
				<div v-if="!emailSent">
					<ValidationObserver ref="emailObserver">
						<ValidationProvider v-slot="{ errors, valid }" name="Email" rules="required|email">
							<b-field :label="$t('Email')" :message="errors"
									 :type="{ 'is-danger': errors[0], 'is-success': valid }">
								<b-input v-model="email" type="email" @input="onEmailInput"></b-input>
							</b-field>
						</ValidationProvider>
					</ValidationObserver>

					<b-button class="mt-4" expanded rounded type="is-primary"
							  @click="sendAuthEmail"
							  :disabled="!canSendEmail"
							  :loading="sendingEmail">
						{{ mode === 'magic-link' ? $t('Send Magic Link') : $t('Send Reset Link') }}
					</b-button>
				</div>

				<!-- Code Input (show after email sent) -->
				<div v-if="emailSent && !verifySuccess">
					<p class="has-text-centered mb-4">
						{{ $t('Check your email for the 6-digit code') }}
					</p>

					<b-field :label="$t('Enter 6-digit code')">
						<b-input v-model="code"
								 type="text"
								 maxlength="6"
								 placeholder="ABC123"
								 :disabled="verifyingCode"
								 :has-counter="false"
								 custom-class="code-input"
								 @input="code = code.toUpperCase()"
								 @keyup.enter.native="verifyCode"></b-input>
					</b-field>

					<b-button class="mt-4" expanded rounded type="is-primary"
							  @click="verifyCode"
							  :disabled="code.length !== 6 || verifyingCode || verifySuccess"
							  :loading="verifyingCode">
						{{ $t('Verify Code') }}
					</b-button>

					<!-- Resend Button -->
					<div class="has-text-centered mt-3">
						<a @click="resendEmail" class="has-text-link is-size-7" style="cursor: pointer;" v-if="!verifyingCode">
							{{ $t('Resend code') }}
						</a>
					</div>
				</div>

				<!-- Success State -->
				<div v-if="verifySuccess" class="has-text-centered">
					<div class="mb-4" style="font-size: 4rem; color: #48c774;">✓</div>
					<h3 class="title is-5">{{ $t('Code verified!') }}</h3>
					<p>{{ $t('Logging you in...') }}</p>
				</div>

				<!-- Back Button -->
				<div class="has-text-centered mt-4">
					<a @click="resetToNormalMode" class="has-text-link is-size-7" style="cursor: pointer;">
						← {{ $t('Back to login') }}
					</a>
				</div>
				</div>
				</transition>
			</div>
		</div>
	</div>
</template>

<script>
import {ValidationObserver, ValidationProvider} from "vee-validate";
import "@/plugins/vee-validate";

export default {

	name: "login-page",
	data() {
		return {
			mode: 'normal', // 'normal', 'magic-link', 'password-reset'
			username: '',
			password: '',
			email: '',
			code: '',
			isLoading: false,
			sendingEmail: false,
			emailSent: false,
			verifyingCode: false,
			verifySuccess: false,
			message: "",
			notificationShow: false,
			notificationType: 'is-danger',
			canSendEmail: false,
		}
	},
	components: {
		ValidationObserver,
		ValidationProvider,
	},
	beforeMount(){
		let userString = localStorage.getItem('user')
		if (userString) {
			let name = JSON.parse(userString).username || '';
			this.username = name;
		}
	},
	mounted() {
		document.querySelector('.modal.is-active ')?.remove();
	},

	methods: {
		async login() {
			try {
				const userRes = await this.$api.users.login(this.username, this.password)
				localStorage.setItem("access_token", userRes.data.data.token.access_token);
				localStorage.setItem("refresh_token", userRes.data.data.token.refresh_token);
				localStorage.setItem("expires_at", userRes.data.data.token.expires_at);
				localStorage.setItem("user", JSON.stringify(userRes.data.data.user));

				this.$store.commit("SET_USER", userRes.data.data.user);
				this.$store.commit("SET_ACCESS_TOKEN", userRes.data.data.token.access_token);
				this.$store.commit("SET_REFRESH_TOKEN", userRes.data.data.token.refresh_token);

				const versionRes = await this.$api.sys.getVersion();
				if (versionRes.data.success == 200) {
					localStorage.setItem("version", versionRes.data.data.current_version);
				}
				this.$router.push("/");
			} catch (err) {
				this.message = this.$t(err.response.data.message)
				this.notificationType = 'is-danger'
				this.notificationShow = true
			}
		},

		onEmailInput() {
			// Validate email has @ symbol
			this.canSendEmail = this.email.includes('@')
		},

		async sendAuthEmail() {
			try {
				this.sendingEmail = true

				if (this.mode === 'magic-link') {
					await this.$api.users.requestMagicLink(this.email)
					this.message = this.$t('Magic link sent to your email')
				} else {
					await this.$api.users.requestPasswordReset(this.email)
					this.message = this.$t('Password reset link sent to your email')
				}

				this.notificationType = 'is-success'
				this.notificationShow = true
				this.emailSent = true
			} catch (err) {
				this.message = err.response?.data?.message || this.$t('Failed to send email')
				this.notificationType = 'is-danger'
				this.notificationShow = true
			} finally {
				this.sendingEmail = false
			}
		},

		async verifyCode() {
			try {
				this.verifyingCode = true

				if (this.mode === 'magic-link') {
					// Verify magic link code and log in
					const res = await this.$api.users.verifyMagicCode(this.email, this.code)

					// Show success state
					this.verifySuccess = true

					// Store tokens and user info
					localStorage.setItem("access_token", res.data.data.token.access_token)
					localStorage.setItem("refresh_token", res.data.data.token.refresh_token)
					localStorage.setItem("expires_at", res.data.data.token.expires_at)
					localStorage.setItem("user", JSON.stringify(res.data.data.user))

					this.$store.commit("SET_USER", res.data.data.user)
					this.$store.commit("SET_ACCESS_TOKEN", res.data.data.token.access_token)
					this.$store.commit("SET_REFRESH_TOKEN", res.data.data.token.refresh_token)

					const versionRes = await this.$api.sys.getVersion()
					if (versionRes.data.success == 200) {
						localStorage.setItem("version", versionRes.data.data.current_version)
					}

					// Wait a moment to show success message, then redirect
					setTimeout(() => {
						this.$router.push("/")
					}, 1000)
				} else {
					// Verify password reset code
					const res = await this.$api.users.verifyPasswordResetCode(this.email, this.code)

					// Show success state
					this.verifySuccess = true

					// Wait a moment, then redirect to password reset page
					setTimeout(() => {
						this.$router.push({
							path: '/auth/password-reset',
							query: {
								token_id: res.data.data.token_id,
								email: res.data.data.email
							}
						})
					}, 800)
				}
			} catch (err) {
				this.message = this.$t('Invalid code. Please try again.')
				this.notificationType = 'is-danger'
				this.notificationShow = true
				this.verifyingCode = false
			}
		},

		async resendEmail() {
			this.code = ''
			this.emailSent = false
			await this.sendAuthEmail()
		},

		resetToNormalMode() {
			this.mode = 'normal'
			this.email = ''
			this.code = ''
			this.emailSent = false
			this.verifySuccess = false
			this.canSendEmail = false
		}
	},

}
</script>

<style lang="scss">
#login-page {
	height: calc(100% - 5.5rem);
	position: relative;
	z-index: 500;

	.login-panel {
		text-align: left;
		background: rgba(255, 255, 255, 0.46);
		backdrop-filter: blur(1rem);
		border-radius: 8px;
		padding: 2.5rem 4rem;

		.label {
			color: #dfdfdf;
		}

		.input {
			background: rgba(255, 255, 255, 0.32);
			border-color: transparent;
		}

		&.step1 {
			padding: 4rem 6rem;
		}

		&.step2 {
			padding: 2.5rem 4rem;
			width: 32rem;
		}

		&.step3 {
			padding: 4rem 8rem;
		}

		&.step4 {
			width: 28rem;
		}
	}

	// Mode container wrapper
	.mode-container {
		position: relative;
	}

	// Override fade-transform for smooth content transition
	// The leaving element is positioned absolutely so it doesn't affect layout
	.fade-transform-leave-active {
		position: absolute;
		width: calc(100% - 8rem); // Account for panel padding
		left: 4rem;
		top: 0;
	}

	// The entering element takes up space normally
	.fade-transform-enter-active {
		position: relative;
	}

	// Code input styling
	.code-input {
		text-transform: uppercase;
		text-align: center;
		font-size: 1.5rem;
		letter-spacing: 0.5rem;
		font-family: monospace;
	}
}

@media screen and (max-width: 480px) {
	.login-panel {
		text-align: left;
		background: rgba(255, 255, 255, 0.46);
		backdrop-filter: blur(1rem);
		border-radius: 8px;
		margin: 0 2rem;
		padding: 2rem !important;

		.label {
			color: #dfdfdf;
		}

		.input {
			background: rgba(255, 255, 255, 0.32);
			border-color: transparent;
		}

		.is-128x128 {
			height: 96px;
			width: 96px;
		}

		.is-3 {
			font-size: 1.5rem;
		}

		&.step1 {
			.is-2 {
				font-size: 1.5rem;
			}

			.subtitle {
				font-size: 1rem;
			}
		}

		&.step3 {
			padding: 4rem !important;
		}
	}
}
</style>