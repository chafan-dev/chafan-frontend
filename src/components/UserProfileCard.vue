<template>
  <div class="c-card pa-3">
    <div class="row">
      <div class="pb-4" :class="{ 'col-12': !$vuetify.breakpoint.mdAndUp }">
        <div v-if="avatarURL" class="px-2" :class="{ 'text-center': !$vuetify.breakpoint.mdAndUp }">
          <router-link :to="`/users/${userPreview.handle}`">
            <v-avatar class="avatarDiv" size="180" tile>
              <v-img :src="avatarURL" alt="Avatar" />
            </v-avatar>
          </router-link>
        </div>
        <UserNameHeadline
          v-if="!$vuetify.breakpoint.mdAndUp"
          :user-preview="userPreview"
          class="text-center"
        />
        <div v-if="follows" class="text-center">
          <div>
            <router-link
              v-if="loggedIn"
              :to="`/users/${userPreview.handle}?tab=followers`"
              class="text-decoration-none"
            >
              有{{ follows.followers_count }}个关注者
            </router-link>
            <span v-else> 有{{ follows.followers_count }}个关注者 </span>
          </div>

          <div>
            <router-link
              v-if="loggedIn"
              :to="`/users/${userPreview.handle}?tab=followed`"
              class="text-decoration-none"
            >
              关注了{{ follows.followed_count }}个人
            </router-link>
            <span v-else> 关注了{{ follows.followed_count }}个人 </span>
          </div>
          <div>
            <span>Karma：{{ userPreview.karma }}</span>
          </div>
          <div class="pt-2">
            <v-btn
              v-if="follows.followed_by_me"
              :disabled="cancelFollowIntermediate"
              class="mt-1 mr-1"
              depressed
              small
              @click="cancelFollow"
            >
              取消关注
              <v-progress-circular v-show="cancelFollowIntermediate" :size="20" indeterminate />
            </v-btn>
            <v-btn
              v-else
              :disabled="followIntermediate"
              class="mt-1 mr-1"
              color="primary"
              depressed
              small
              @click="follow"
            >
              关注
              <v-progress-circular
                v-show="followIntermediate"
                :size="20"
                color="primary"
                indeterminate
              />
            </v-btn>

            <v-btn
              :disabled="privateMessageIntermediate"
              class="mt-1 mr-1"
              depressed
              small
              @click="privateMessage"
            >
              私信
            </v-btn>
          </div>
        </div>
      </div>
      <div class="col align-self-center">
        <UserNameHeadline v-if="$vuetify.breakpoint.mdAndUp" :user-preview="userPreview" />
        <v-divider class="my-2" />
        <UserProfileDetails v-if="userPublic" :user-public="userPublic" />

        <div class="d-flex">
          <span class="text-caption grey--text mt-2">
            这个主页被浏览了{{ userPublic.profile_view_times }}次
          </span>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { IUserFollows, IUserPreview, IUserPublic } from '@/interfaces';
import { api } from '@/api';
import { Component, Vue, Prop } from 'vue-property-decorator';
import { dispatchCaptureApiError } from '@/store/main/actions';
import { apiMe } from '@/api/me';
import { commitSetShowLoginPrompt } from '@/store/main/mutations';
import { readIsLoggedIn, readUserProfile, readToken } from '@/store/main/getters';
import UserProfileDetails from '@/components/UserProfileDetails.vue';
import UserNameHeadline from '@/components/UserNameHeadline.vue';
@Component({
  components: { UserNameHeadline, UserProfileDetails },
})
export default class UserProfileCard extends Vue {
  @Prop() public readonly userPreview!: IUserPreview;
  @Prop() public readonly userPublic: IUserPublic | undefined;
  @Prop() private readonly siteKarmas: number | undefined;
  private loading = true;
  private follows: IUserFollows | null = null;
  private followIntermediate = false;
  private cancelFollowIntermediate = false;
  private privateMessageIntermediate = false;
  private avatarURL: string | null = null;

  get currentUserId() {
    return readUserProfile(this.$store)?.uuid;
  }

  get loggedIn() {
    return readIsLoggedIn(this.$store);
  }

  get token() {
    return readToken(this.$store);
  }

  private shortIntro(intro: string) {
    if (!intro) {
      return intro;
    }
    if (intro.length > 25) {
      return intro.substring(0, 25) + '...';
    }
    return intro;
  }

  private async mounted() {
    if (this.userPublic && this.userPublic.gif_avatar_url) {
      this.avatarURL = this.userPublic.gif_avatar_url;
    } else if (this.userPreview.avatar_url) {
      this.avatarURL = this.userPreview.avatar_url;
    } else {
      this.avatarURL = '/img/default-avatar.png';
    }
    await dispatchCaptureApiError(this.$store, async () => {
      this.follows = (await apiMe.getUserFollows(this.token, this.userPreview.uuid)).data;
    });
    this.loading = false;
  }

  private async follow() {
    if (!this.currentUserId) {
      commitSetShowLoginPrompt(this.$store, true);
      return;
    }
    this.followIntermediate = true;
    if (this.userPreview !== null) {
      this.follows = (await apiMe.followUser(this.token, this.userPreview.uuid)).data;
      this.followIntermediate = false;
    }
  }

  private async cancelFollow() {
    if (this.userPreview !== null) {
      this.cancelFollowIntermediate = true;
      await dispatchCaptureApiError(this.$store, async () => {
        this.follows = (await apiMe.cancelFollowUser(this.token, this.userPreview.uuid)).data;
        this.cancelFollowIntermediate = false;
      });
    }
  }

  private async privateMessage() {
    if (!this.currentUserId) {
      commitSetShowLoginPrompt(this.$store, true);
      return;
    }
    this.privateMessageIntermediate = true;
    await dispatchCaptureApiError(this.$store, async () => {
      const r0 = await api.createChannel(this.token, {
        private_with_user_uuid: this.userPreview.uuid,
      });
      const channelId = r0.data.id;
      await this.$router.push(`/channels/${channelId}`);
    });
  }
}
</script>
