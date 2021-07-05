<template>
  <div>
    <div v-if="userPublic.about">
      <span class="subheading secondary--text text--lighten-3">关于我：</span>
      <Viewer :body="userPublic.about" :editor="userPublic.about_editor" />
    </div>

    <div
      v-if="
        userPublic.homepage_url ||
        userPublic.github_username ||
        userPublic.twitter_username ||
        userPublic.linkedin_url
      "
      class="my-3"
    >
      <span class="subheading secondary--text text--lighten-3"> 链接： </span>
      <a
        v-if="userPublic.homepage_url"
        :href="userPublic.homepage_url"
        class="text-decoration-none"
        target="_blank"
      >
        <WebIcon />
        个人主页
      </a>
      <a
        v-if="userPublic.github_username"
        :href="canonicalURLfromUsername(userPublic.github_username, 'github.com')"
        class="text-decoration-none"
        target="_blank"
      >
        <GithubIcon />
        Github
      </a>
      <a
        v-if="userPublic.twitter_username"
        :href="canonicalURLfromUsername(userPublic.twitter_username, 'twitter.com')"
        class="text-decoration-none"
        target="_blank"
      >
        <TwitterIcon />
        Twitter
      </a>
      <a
        v-if="userPublic.linkedin_url"
        :href="userPublic.linkedin_url"
        class="text-decoration-none"
        target="_blank"
      >
        <LinkedinIcon />
        Linkedin
      </a>
    </div>

    <template v-if="userPublic.profession_topic">
      <div v-if="full" class="my-3">
        <div class="subheading secondary--text text--lighten-3">所在行业</div>
        <div class="title primary--text text--darken-2">
          <v-chip :to="'/topics/' + userPublic.profession_topic.uuid"
            >{{ userPublic.profession_topic.name }}
          </v-chip>
        </div>
      </div>
      <div v-else>
        所在行业：<span>{{ userPublic.profession_topic.name }}</span>
      </div>
    </template>

    <template v-if="eduExps && eduExps.length > 0">
      <div v-if="full" class="my-3">
        <div class="subheading secondary--text text--lighten-3">教育经历</div>
        <div
          v-for="(eduExp, index) in eduExps"
          :key="index"
          class="title primary--text text--darken-2"
        >
          <router-link :to="'/topics/' + eduExp.school_topic.uuid" class="text-decoration-none">
            {{ eduExp.school_topic.name }}
          </router-link>
          ({{ $t(eduExp.level) }})
        </div>
      </div>
      <div v-else>教育经历：{{ eduExps[0].school_topic.name }} 等</div>
    </template>
    <v-skeleton-loader v-else-if="eduExps === null && loggedIn" type="text" />

    <template v-if="workExps && workExps.length > 0">
      <div v-if="full" class="my-3">
        <div class="subheading secondary--text text--lighten-3">工作经历</div>
        <div
          v-for="(workExp, index) in workExps"
          :key="index"
          class="title primary--text text--darken-2"
        >
          <router-link :to="'/topics/' + workExp.position_topic.uuid" class="text-decoration-none">
            {{ workExp.position_topic.name }}
          </router-link>
          @
          <router-link :to="'/topics/' + workExp.company_topic.uuid" class="text-decoration-none">
            {{ workExp.company_topic.name }}
          </router-link>
        </div>
      </div>
      <div v-else>工作经历：{{ workExps[0].company_topic.name }} 等</div>
    </template>
    <v-skeleton-loader v-else-if="workExps === null && loggedIn" type="text" />

    <template v-if="full">
      <div v-if="userPublic.residency_topics.length > 0" class="my-3">
        <div class="subheading secondary--text text--lighten-3">居住过的地方</div>
        <div>
          <v-chip-group :column="!$vuetify.breakpoint.mobile">
            <v-chip
              v-for="topic in userPublic.residency_topics"
              :key="topic.uuid"
              :to="'/topics/' + topic.uuid"
              >{{ topic.name }}
            </v-chip>
          </v-chip-group>
        </div>
      </div>

      <template v-if="sites !== null">
        <div v-if="sites.length > 0">
          <div
            v-if="$vuetify.breakpoint.mdAndUp"
            class="subheading secondary--text text--lighten-3"
          >
            加入的圈子：
            <span>
              <SiteBtn v-for="site in sites" :key="site.uuid" :site="site" />
            </span>
          </div>
          <div v-else>
            <span class="subheading secondary--text text--lighten-3 mr-2"> 加入的圈子： </span>
            <SiteBtn v-for="site in sites" :key="site.uuid" :site="site" />
          </div>
        </div>
      </template>
      <v-skeleton-loader v-else-if="loggedIn" type="text" />

      <div v-if="userPublic.subscribed_topics.length > 0" class="my-3">
        <div class="subheading secondary--text text--lighten-3">关注的话题：</div>
        <v-chip
          v-for="topic in userPublic.subscribed_topics"
          :key="topic.uuid"
          :to="'/topics/' + topic.uuid"
          class="mr-1 mb-1"
          small
          >{{ topic.name }}
        </v-chip>
      </div>
    </template>

    <div class="d-flex">
      <div v-if="enableFull" class="mr-1">
        <a @click="full = !full">
          <span v-if="full"> 收起 </span>
          <span v-else> ...展开更多 </span>
        </a>
      </div>
      <v-spacer />
      <v-btn
        depressed
        v-if="currentUserId === userPublic.uuid"
        color="primary"
        small
        to="/profile/edit"
      >
        编辑
      </v-btn>
    </div>
  </div>
</template>

<script lang="ts">
import { ISite, IUserEducationExperience, IUserPublic, IUserWorkExperience } from '@/interfaces';
import { Component, Vue, Prop } from 'vue-property-decorator';
import { readIsLoggedIn, readToken, readUserProfile } from '@/store/main/getters';
import SiteBtn from '@/components/SiteBtn.vue';
import TwitterIcon from '@/components/icons/TwitterIcon.vue';
import WebIcon from '@/components/icons/WebIcon.vue';
import GithubIcon from '@/components/icons/GithubIcon.vue';
import LinkedinIcon from '@/components/icons/LinkedinIcon.vue';
import { apiPeople } from '@/api/people';
import { apiSite } from '@/api/site';

@Component({
  components: {
    SiteBtn,
    TwitterIcon,
    WebIcon,
    GithubIcon,
    LinkedinIcon,
  },
})
export default class UserProfileDetails extends Vue {
  @Prop() public readonly userPublic!: IUserPublic;
  private eduExps: IUserEducationExperience[] | null = null;
  private workExps: IUserWorkExperience[] | null = null;
  private sites: ISite[] | null = null;
  private full: boolean = false;
  private enableFull: boolean = false;

  get currentUserId() {
    return readUserProfile(this.$store)?.uuid;
  }

  async mounted() {
    if (this.loggedIn) {
      this.eduExps = (
        await apiPeople.getUserEducationExperiences(this.token, this.userPublic.uuid)
      ).data;
      this.workExps = (
        await apiPeople.getUserWorkExperiences(this.token, this.userPublic.uuid)
      ).data;
      const responses = await Promise.all(
        this.userPublic.profiles.map((p) => apiSite.getSite(p.site.subdomain))
      );
      this.sites = responses.map((r) => r.data);
      this.enableFull =
        this.userPublic.subscribed_topics.length > 0 ||
        this.userPublic.residency_topics.length > 0 ||
        this.sites?.length > 0;
    }
  }

  get loggedIn() {
    return readIsLoggedIn(this.$store);
  }

  get token() {
    return readToken(this.$store);
  }

  canonicalURLfromUsername(username: string, domain: string) {
    if (username.startsWith(domain)) {
      return 'https://' + username;
    } else if (username.includes(domain)) {
      return username;
    } else {
      return 'https://' + domain + '/' + username;
    }
  }
}
</script>
