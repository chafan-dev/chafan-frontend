<template>
  <v-main>
    <v-container fill-height fluid>
      <v-layout align-center justify-center>
        <v-flex md6 sm8 xs12>
          <ValidationObserver v-slot="{ handleSubmit, valid }">
            <v-card class="elevation-12">
              <v-toolbar color="primary" dark>
                <v-toolbar-title>{{ appName }} 注册</v-toolbar-title>
              </v-toolbar>
              <v-card-text>
                <div class="black--text mb-4">
                  注册前请仔细阅读<a
                    class="text-decoration-none"
                    href="https://about.cha.fan/docs"
                    target="_blank"
                    >本网站相关文档</a
                  >。 如果你继续注册使用本网站，将视为同意<a
                    class="text-decoration-none"
                    href="https://about.cha.fan/docs/user-agreement/"
                    target="_blank"
                    >「网站用户协议」</a
                  >
                  并了解<a
                    class="text-decoration-none"
                    href="https://about.cha.fan/docs/code-of-conduct/"
                    target="_blank"
                    >「社区行为守则」</a
                  >的内容。
                </div>
                <v-form autocomplete="off">
                  <h3>请先填入邀请码（20位）</h3>
                  <ValidationProvider v-slot="{ errors }" name="邀请码" rules="required">
                    <v-text-field v-model="invitationToken" name="邀请码" required type="text" />
                    <span class="error--text">{{ errors[0] }}</span>
                  </ValidationProvider>

                  <v-divider class="mt-2 mb-4" />

                  <h3>注册信息</h3>

                  <ValidationProvider v-slot="{ errors }" name="email" rules="required|email">
                    <v-text-field v-model="email" :label="$t('Email')" name="login" type="text">
                      <template v-slot:prepend>
                        <AccountIcon />
                      </template>
                    </v-text-field>
                    <span class="error--text">{{ errors[0] }}</span>
                  </ValidationProvider>

                  <v-text-field
                    v-model="verificationCode"
                    label="验证码"
                    name="verification-code"
                    type="text"
                  >
                    <template v-slot:prepend>
                      <VerifyCodeIcon />
                    </template>
                    <template v-slot:append-outer>
                      <v-btn
                        :disabled="intermediate || verificationCodeSent"
                        color="primary"
                        depressed
                        small
                        @click="sendVerificationCode"
                      >
                        发送验证码
                      </v-btn>
                    </template>
                  </v-text-field>

                  <ValidationProvider v-slot="{ errors }" name="password" rules="required|password">
                    <v-text-field
                      v-model="password"
                      :label="$t('Password')"
                      autocomplete="new-password"
                      name="new-password"
                      required
                      type="password"
                    >
                      <template v-slot:prepend>
                        <PasswordIcon />
                      </template>
                    </v-text-field>
                    <span class="error--text">{{ errors[0] }}</span>
                  </ValidationProvider>

                  <ValidationProvider v-slot="{ errors }" name="handle" rules="required">
                    <v-text-field
                      v-model="handle"
                      :label="$t('Unique username (you can change this later)')"
                      name="handle"
                      required
                      type="text"
                    >
                      <template v-slot:prepend>
                        <HandleIcon />
                      </template>
                    </v-text-field>
                    <span class="error--text">{{ errors[0] }}</span>
                  </ValidationProvider>
                </v-form>
              </v-card-text>
              <v-card-actions>
                <v-spacer />
                <v-btn
                  :disabled="intermediate || !valid"
                  color="primary"
                  depressed
                  small
                  @click="handleSubmit(openAccount)"
                >
                  验证并注册
                </v-btn>
              </v-card-actions>
            </v-card>
          </ValidationObserver>
        </v-flex>
      </v-layout>
    </v-container>
  </v-main>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import { api } from '@/api';
import { appName } from '@/env';
import { commitAddNotification, commitSetShowLoginPrompt } from '@/store/main/mutations';
import AccountIcon from '@/components/icons/AccountIcon.vue';
import PasswordIcon from '@/components/icons/PasswordIcon.vue';
import HandleIcon from '@/components/icons/HandleIcon.vue';
import VerifyCodeIcon from '@/components/icons/VerifyCodeIcon.vue';

import { dispatchCaptureApiError } from '@/store/main/actions';

// TODO: share a parent component with Login.vue
@Component({
  components: {
    AccountIcon,
    PasswordIcon,
    HandleIcon,
    VerifyCodeIcon,
  },
})
export default class Signup extends Vue {
  private email: string = '';
  private password: string = '';
  private handle: string = '';
  private appName = appName;
  private verificationCode: string = '';
  private invitationToken: string = '';

  private intermediate = false;
  private verificationCodeSent = false;

  private mounted() {
    commitSetShowLoginPrompt(this.$store, false);
    if (this.$route.query.invitation_link_uuid) {
      this.invitationToken = this.$route.query.invitation_link_uuid.toString();
    }
  }

  private async sendVerificationCode() {
    this.intermediate = true;
    await dispatchCaptureApiError(this.$store, async () => {
      const response = await api.sendVerificationCode({ email: this.email });
      if (response) {
        const msg =
          response.data.msg === 'Verification code email sent'
            ? '验证码已发送到邮箱'
            : response.data.msg;
        commitAddNotification(this.$store, {
          content: msg,
          color: 'success',
        });
      }
      this.verificationCodeSent = true;
      this.intermediate = false;
    });
  }

  private async openAccount() {
    await dispatchCaptureApiError(this.$store, async () => {
      this.intermediate = true;
      const response = await api.openAccount(
        this.email,
        this.handle,
        this.verificationCode,
        this.password,
        this.invitationToken
      );
      if (response) {
        commitAddNotification(this.$store, {
          content: '账号创建成功，返回登陆界面',
          color: 'success',
        });
        this.$router.push('/login');
      }
      this.intermediate = false;
    });
  }
}
</script>
