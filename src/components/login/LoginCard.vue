<template>
  <v-sheet v-bind="$attrs">
    <v-row align="center" class="fill-height" no-gutters>
      <v-col v-if="$vuetify.breakpoint.lgAndUp && showTopBar" cols="12" md="6">
        <v-sheet color="primary darken-2" dark height="100vh" width="100%">
          <v-container class="justify-center" fill-height>
            <div class="d-flex align-center flex-column text-center">
              <h1 class="text-xl-h1 text-md-h2 font-weight-bold">
                {{ appName }}
              </h1>

              <h3 class="text-xl-h3 text-md-h4 mt-4">有深度的社交问答网站</h3>
            </div>
          </v-container>
        </v-sheet>
      </v-col>

      <v-col :lg="showTopBar ? 6 : 12" cols="12">
        <v-container>
          <v-row justify="center">
            <v-col
              :lg="showTopBar ? 6 : 11"
              :md="showTopBar ? 8 : 11"
              :xl="showTopBar ? 5 : 11"
              cols="12"
            >
              <ValidationObserver v-slot="{ handleSubmit, valid }">
                <h3 class="text-h3 font-weight-bold text-center">登录</h3>

                <div class="mt-4">
                  <!-- NOTE(zhen): v-if doesn't work on ValidationProvider -->
                  <template v-if="loginMethod === 'email'">
                    <ValidationProvider v-slot="{ errors }" name="email" rules="email">
                      <v-text-field
                        v-model="email"
                        label="邮箱地址"
                        name="login"
                        type="text"
                        @keyup.enter="submit"
                      >
                        <template v-slot:prepend>
                          <AccountCircleOutlineIcon />
                        </template>
                      </v-text-field>
                      <span class="error--text">{{ errors[0] }}</span>
                    </ValidationProvider>

                    <ValidationProvider v-slot="{ errors }" name="password" rules="required">
                      <v-text-field
                        id="password"
                        v-model="password"
                        label="密码"
                        name="password"
                        required
                        type="password"
                        @keyup.enter="submit"
                      >
                        <template v-slot:prepend>
                          <LockOutlineIcon />
                        </template>
                      </v-text-field>
                      <span class="error--text">{{ errors[0] }}</span>
                    </ValidationProvider>

                    <div class="text-center" v-if="enableCaptcha">
                      <vue-hcaptcha :sitekey="hCaptchaSiteKey" @verify="verifiedCaptchaToken" />
                    </div>

                    <v-sheet class="mb-2">
                      <div v-if="loginError">
                        <v-alert :value="loginError" transition="fade-transition" type="error">
                          邮箱地址或密码不正确
                        </v-alert>
                      </div>
                    </v-sheet>
                  </template>

                  <v-sheet v-else-if="loginMethod === 'cellphone'">
                    <ValidationProvider
                      v-slot="{ errors }"
                      name="phonenumber"
                      rules="phone_number_e164"
                    >
                      <v-text-field
                        v-model="phoneNumber"
                        label="手机号"
                        name="phonenumber"
                        type="text"
                        @keyup.enter="submit"
                      >
                        <template v-slot:prepend>
                          <CellphoneIcon />
                        </template>
                      </v-text-field>
                      <span class="error--text">{{ errors[0] }}</span>
                    </ValidationProvider>

                    <div class="d-flex">
                      <v-text-field
                        v-model="verificationCode"
                        label="验证码"
                        name="verification-code"
                        type="text"
                      >
                        <template v-slot:prepend>
                          <VerifyCodeIcon />
                        </template>
                      </v-text-field>

                      <v-btn
                        :disabled="verificationCodeDisabled"
                        class="mt-4"
                        color="primary"
                        depressed
                        small
                        @click="sendVerificationCode"
                      >
                        发送验证码
                      </v-btn>
                    </div>
                  </v-sheet>

                  <v-sheet>
                    <v-btn
                      :disabled="
                        submitIntermediate ||
                        (loginMethod === 'cellphone' && !verificationCode) ||
                        (loginMethod === 'email' && enableCaptcha && !captchaToken) ||
                        !valid
                      "
                      block
                      color="primary"
                      depressed
                      large
                      @click.prevent="handleSubmit(submit)"
                      class="white--text"
                    >
                      登录
                      <v-progress-circular v-show="submitIntermediate" :size="20" indeterminate />
                    </v-btn>
                  </v-sheet>

                  <v-sheet class="mt-4 d-flex align-center justify-end">
                    <v-btn
                      class="text-capitalize"
                      depressed
                      href="/recover-password"
                      link
                      small
                      text
                    >
                      <HelpIcon class="mr-1" />
                      忘记密码？
                    </v-btn>

                    <v-btn
                      class="text-capitalize"
                      depressed
                      href="https://about.cha.fan/signup"
                      link
                      small
                      text
                    >
                      <HelpCircleOutline class="mr-1" />
                      如何注册「茶饭」账号?
                    </v-btn>

                    <v-btn
                      class="text-capitalize"
                      depressed
                      href="mailto:contact@cha.fan"
                      link
                      small
                      text
                    >
                      <EmailEditOutline class="mr-1" />
                      联系我们
                    </v-btn>
                  </v-sheet>
                </div>

                <div class="my-8">
                  <div class="d-flex align-center">
                    <v-sheet color="grey lighten-2" height=".5" width="100%" />

                    <div class="mx-2">Or</div>

                    <v-sheet color="grey lighten-2" height=".5" width="100%" />
                  </div>
                </div>

                <div class="text-center">
                  <v-btn class="text-capitalize" depressed large @click="switchLoginMethod">
                    使用
                    <template v-if="loginMethod === 'email'"> 手机号 </template>
                    <template v-else> 邮箱 </template>
                    登录
                  </v-btn>
                </div>
              </ValidationObserver>
            </v-col>
          </v-row>
        </v-container>
      </v-col>
    </v-row>
  </v-sheet>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';
import AccountIcon from '@/components/icons/AccountIcon.vue';
import PasswordIcon from '@/components/icons/PasswordIcon.vue';
import JoinChafanIcon from '@/components/icons/JoinChafanIcon.vue';
import EmailIcon from '@/components/icons/EmailIcon.vue';
import CellphoneIcon from '@/components/icons/CellphoneIcon.vue';
import VerifyCodeIcon from '@/components/icons/VerifyCodeIcon.vue';

import { appName, enableCaptcha } from '@/env';
import { readLoginError } from '@/store/main/getters';
import { dispatchCaptureApiError, dispatchLogIn } from '@/store/main/actions';
import { commitAddNotification } from '@/store/main/mutations';
import { api } from '@/api';
import { captureException } from '@sentry/vue';
import AccountCircleOutlineIcon from '@/components/icons/AccountCircleOutlineIcon.vue';
import LockOutline from '@/components/icons/LockOutlineIcon.vue';
import LockOutlineIcon from '@/components/icons/LockOutlineIcon.vue';
import HelpCircleOutline from '@/components/icons/HelpCircleOutline.vue';
import EmailEditOutline from '@/components/icons/EmailEditOutline.vue';
import { hCaptchaSiteKey } from '@/env';
import VueHcaptcha from '@hcaptcha/vue-hcaptcha';
import HelpIcon from '@/components/icons/HelpIcon.vue';

@Component({
  components: {
    HelpIcon,
    EmailEditOutline,
    HelpCircleOutline,
    LockOutlineIcon,
    LockOutline,
    AccountCircleOutlineIcon,
    AccountIcon,
    PasswordIcon,
    VerifyCodeIcon,
    JoinChafanIcon,
    EmailIcon,
    CellphoneIcon,
    VueHcaptcha,
  },
})
export default class LoginCard extends Vue {
  @Prop({ default: true }) public readonly showTopBar!: boolean;
  private email: string = '';
  private phoneNumber: string = '';
  private password: string = '';
  private appName = appName;
  private loginMethod: 'email' | 'cellphone' = 'email';
  private submitIntermediate = false;
  private sendVerificationCodeIntermediate = false;
  private verificationCodeDisabled = false;
  private verificationCode: string | null = null;
  private captchaVerified = false;
  private captchaToken: string | null = null;

  get enableCaptcha() {
    return enableCaptcha;
  }

  get hCaptchaSiteKey() {
    return hCaptchaSiteKey;
  }

  private get loginError() {
    this.submitIntermediate = false;
    return readLoginError(this.$store);
  }

  private async sendVerificationCode() {
    if (!this.phoneNumber) {
      commitAddNotification(this.$store, {
        content: '电话号码为空',
        color: 'error',
      });
      return;
    }
    this.sendVerificationCodeIntermediate = true;
    await dispatchCaptureApiError(this.$store, async () => {
      const response = await api.sendVerificationCode({
        phone_number: this.phoneNumber!,
      });
      if (response) {
        commitAddNotification(this.$store, {
          content: response.data.msg,
          color: 'success',
        });
        this.verificationCodeDisabled = true;
      }
      this.sendVerificationCodeIntermediate = false;
    });
  }

  private async submit() {
    this.submitIntermediate = true;
    try {
      if (this.loginMethod === 'email') {
        if (this.enableCaptcha && !this.captchaToken) {
          this.submitIntermediate = false;
          return;
        }
        await dispatchLogIn(this.$store, {
          type: this.loginMethod,
          username: this.email,
          password: this.password,
          hcaptcha_token: this.enableCaptcha && this.captchaToken ? this.captchaToken : undefined,
        });
      } else {
        if (this.verificationCode) {
          await dispatchLogIn(this.$store, {
            type: this.loginMethod,
            phoneNumber: this.phoneNumber,
            code: this.verificationCode,
          });
        }
      }
    } catch (err) {
      if (err.toString() === 'Error: Incorrect email or password') {
        commitAddNotification(this.$store, {
          content: this.$t('Incorrect email or password').toString(),
          color: 'error',
        });
      } else {
        captureException(err);
      }
    } finally {
      this.submitIntermediate = false;
    }
  }

  private async switchLoginMethod() {
    this.loginMethod = this.loginMethod === 'email' ? 'cellphone' : 'email';
  }

  private verifiedCaptchaToken(token: string) {
    this.captchaVerified = true;
    this.captchaToken = token;
  }
}
</script>
