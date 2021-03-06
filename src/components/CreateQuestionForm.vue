<template>
  <v-card>
    <v-card-title v-if="showTitle" class="primary--text headline">
      {{ $t('提问') }}
    </v-card-title>
    <v-card-text>
      <v-autocomplete
        v-if="site === undefined"
        v-model="selectedSite"
        :items="siteProfiles"
        :label="$t('Circle') + $t(' (加入后在此处可见, 「大广场」不限话题)')"
        item-text="site.name"
        item-value="site"
      />
      <v-textarea v-model="newQuestionTitle" :label="$t('Title')" auto-grow dense rows="3" />
      <div class="d-flex">
        <v-spacer />
        <span class="text-caption grey--text">
          {{ $t('创建后添加细节') }}
        </span>
      </div>
    </v-card-text>
    <v-card-actions>
      <v-spacer />
      <v-btn :disabled="intermediate" color="primary" small depressed @click="postNewQuestion">
        {{ $t('提交新问题') }}
        <v-progress-circular v-if="intermediate" indeterminate size="20" />
      </v-btn>
    </v-card-actions>
  </v-card>
</template>

<script lang="ts">
import { Component, Vue, Prop } from 'vue-property-decorator';
import { commitAddNotification, commitSetShowLoginPrompt } from '@/store/main/mutations';
import { ISite, IUserSiteProfile } from '@/interfaces';
import UserLink from '@/components/UserLink.vue';
import Invite from '@/components/Invite.vue';
import { dispatchCaptureApiError } from '@/store/main/actions';
import { apiMe } from '@/api/me';
import { apiQuestion } from '@/api/question';
import { readToken } from '@/store/main/getters';

@Component({
  components: { UserLink, Invite },
})
export default class CreateQuestionForm extends Vue {
  @Prop() private readonly site: ISite | undefined;
  @Prop({ default: false }) private readonly showTitle!: boolean;

  private newQuestionTitle: string = '';
  private intermediate = false;
  private siteProfiles: IUserSiteProfile[] = [];
  private selectedSite: ISite | null = null;

  get token() {
    return readToken(this.$store);
  }

  private async mounted() {
    // FIXME: Cache user's site profile on the client side
    if (this.site === undefined && this.token) {
      await dispatchCaptureApiError(this.$store, async () => {
        this.siteProfiles = (await apiMe.getUserSiteProfiles(this.token)).data;
      });
    }
  }

  private async postNewQuestion() {
    if (!this.token) {
      commitSetShowLoginPrompt(this.$store, true);
      return;
    }
    let siteUUID;
    if (this.site) {
      siteUUID = this.site.uuid;
    } else if (this.selectedSite) {
      siteUUID = this.selectedSite.uuid;
    } else {
      commitAddNotification(this.$store, {
        content: this.$t('所选圈子不能为空').toString(),
        color: 'error',
      });
      return;
    }
    if (this.newQuestionTitle.length < 5) {
      commitAddNotification(this.$store, {
        content: this.$t('Title is too short: minimum length 5 characters.').toString(),
        color: 'error',
      });
      return;
    }
    await dispatchCaptureApiError(this.$store, async () => {
      this.intermediate = true;
      const response = await apiQuestion.postQuestion(this.token, {
        site_uuid: siteUUID,
        title: this.newQuestionTitle,
      });
      this.intermediate = false;
      if (response) {
        try {
          localStorage.setItem('new-question', 'true');
        } catch (e) {}
        this.$router.push(`/questions/${response.data.uuid}`);
      }
    });
  }
}
</script>
