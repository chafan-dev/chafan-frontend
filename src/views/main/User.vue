<template>
  <v-container fluid>
    <v-row justify="center">
      <!-- Main panel -->
      <v-col :class="{ 'col-8': $vuetify.breakpoint.mdAndUp }" fluid>
        <UserProfileCard
          v-if="userPublic || userPublicForVisitor"
          :userPreview="userPublic ? userPublic : userPublicForVisitor"
          :userPublic="userPublic ? userPublic : userPublicForVisitor"
          class="mb-4"
        />
        <v-skeleton-loader v-else type="card" />

        <div v-if="loggedIn">
          <v-tabs v-model="currentTabItem" :fixed-tabs="$vuetify.breakpoint.mdAndUp" show-arrows>
            <v-tab v-for="item in tabItems" :key="item.code" :href="'#' + item.code">
              {{ $t(item.text) }}
              <span v-if="item.tabExtraCount && userPublic" class="ml-1">
                ({{ item.tabExtraCount(userPublic) }})
              </span>
            </v-tab>

            <v-tab-item class="pt-2" value="recent">
              <UserFeed
                v-if="userPublic"
                :enable-show-explore-sites="false"
                :subject-user-uuid="userPublic.uuid"
                :user-profile="userProfile"
              />
            </v-tab-item>

            <v-tab-item class="pt-2" value="answers">
              <DynamicItemList
                v-if="userPublic"
                v-slot="{ item }"
                :loadItems="loadAnswers"
                emptyItemsText="暂无"
                nullItemsText=""
              >
                <Answer :answerPreview="item" />
              </DynamicItemList>
            </v-tab-item>

            <v-tab-item class="pt-2" value="questions">
              <DynamicItemList
                v-if="userPublic"
                v-slot="{ item }"
                :loadItems="loadQuestions"
                emptyItemsText="暂无"
                nullItemsText=""
              >
                <QuestionPreview :questionPreview="item" />
              </DynamicItemList>
            </v-tab-item>

            <v-tab-item class="pt-2" value="articles">
              <DynamicItemList
                v-if="userPublic"
                v-slot="{ item }"
                :loadItems="loadArticles"
                emptyItemsText="暂无"
                nullItemsText=""
              >
                <ArticlePreview :articlePreview="item" />
              </DynamicItemList>
            </v-tab-item>

            <v-tab-item class="pt-2" value="submissions">
              <DynamicItemList
                v-if="userPublic"
                v-slot="{ item }"
                :loadItems="loadSubmissions"
                emptyItemsText="暂无"
                nullItemsText=""
              >
                <SubmissionCard :submission="item" />
              </DynamicItemList>
            </v-tab-item>

            <v-tab-item value="followers">
              <div v-if="followers !== null" class="my-4">
                <EmptyPlaceholder v-if="followers.length === 0" />
                <UserGrid :users="followers" />
              </div>
              <v-skeleton-loader v-else type="paragraph" />
            </v-tab-item>

            <v-tab-item value="followed">
              <div v-if="followed !== null" class="my-4">
                <EmptyPlaceholder v-if="followed.length === 0" />
                <UserGrid :users="followed" />
              </div>
              <v-skeleton-loader v-else type="paragraph" />
            </v-tab-item>
          </v-tabs>
        </div>
        <div v-else class="ml-7 mr-7">
          <div class="text-center">
            <RegisteredUserOnlyIcon class="mr-2" />
            <span>登录后查看更多</span>
          </div>
        </div>
      </v-col>
    </v-row>
  </v-container>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import { apiPeople } from '@/api/people';

import {
  IAnswerPreview,
  IArticlePreview,
  IQuestionPreview,
  ISubmission,
  IUserPreview,
  IUserPublic,
  IUserPublicForVisitor,
} from '@/interfaces';
import QuestionPreview from '@/components/question/QuestionPreview.vue';
import ArticlePreview from '@/components/ArticlePreview.vue';
import Answer from '@/components/Answer.vue';
import UserProfileCard from '@/components/UserProfileCard.vue';

import SiteBtn from '@/components/SiteBtn.vue';
import SubmissionCard from '@/components/SubmissionCard.vue';
import UserLink from '@/components/UserLink.vue';
import UserGrid from '@/components/UserGrid.vue';
import DynamicItemList from '@/components/DynamicItemList.vue';

import { dispatchCaptureApiError } from '@/store/main/actions';
import { readIsLoggedIn, readToken, readUserProfile } from '@/store/main/getters';
import RegisteredUserOnlyIcon from '@/components/icons/RegisteredUserOnlyIcon.vue';
import { Route, RouteRecord } from 'vue-router';
import { isEqual, updateHead } from '@/common';
import UserFeed from '@/components/home/UserFeed.vue';
import EmptyPlaceholder from '@/components/EmptyPlaceholder.vue';

@Component({
  components: {
    EmptyPlaceholder,
    UserFeed,
    RegisteredUserOnlyIcon,
    QuestionPreview,
    Answer,
    UserProfileCard,
    UserLink,
    SiteBtn,
    UserGrid,
    ArticlePreview,
    SubmissionCard,
    DynamicItemList,
  },
})
export default class User extends Vue {
  private tabItems = [
    {
      code: 'recent',
      text: 'Recent',
    },
    {
      code: 'answers',
      text: 'Answers',
      tabExtraCount: (userPublic: IUserPublic) => userPublic.answers_count,
    },
    {
      code: 'questions',
      text: 'Questions',
      tabExtraCount: (userPublic: IUserPublic) => userPublic.questions_count,
    },
    {
      code: 'articles',
      text: 'Articles',
      tabExtraCount: (userPublic: IUserPublic) => userPublic.articles_count,
    },
    {
      code: 'submissions',
      text: 'Sharings',
      tabExtraCount: (userPublic: IUserPublic) => userPublic.submissions_count,
    },
    {
      code: 'followers',
      text: 'Followers',
      render: this.renderFollowers,
    },
    {
      code: 'followed',
      text: 'Followed',
      render: this.renderFollowed,
    },
  ];
  private userPublic: IUserPublic | null = null;
  private userPublicForVisitor: IUserPublicForVisitor | null = null;
  private followers: IUserPreview[] | null = null;
  private followed: IUserPreview[] | null = null;

  get loggedIn() {
    return readIsLoggedIn(this.$store);
  }

  get currentUserId() {
    return readUserProfile(this.$store)?.uuid;
  }

  get userProfile() {
    return readUserProfile(this.$store);
  }

  get handle() {
    return this.$route.params.handle;
  }

  get currentTabItem() {
    return this.$route.query.tab ? this.$route.query.tab : 'profile';
  }

  set currentTabItem(tab) {
    if (tab !== 'profile') {
      this.$router.replace({ query: { ...this.$route.query, tab } });
    } else {
      this.$router.replace({ query: { ...this.$route.query, tab: undefined } });
    }
  }

  get token() {
    return readToken(this.$store);
  }

  beforeRouteUpdate(to: Route, from: Route, next: () => void) {
    next();
    const matched = from.matched.find((record: RouteRecord) => record.name === 'user');
    if (matched && !isEqual(to.params, from.params)) {
      this.userPublic = null;
      this.userPublicForVisitor = null;
      this.followers = null;
      this.followed = null;
      this.load();
    }
  }

  public async mounted() {
    await this.load();
  }

  private async load() {
    await dispatchCaptureApiError(this.$store, async () => {
      if (this.loggedIn) {
        this.userPublic = (await apiPeople.getUserPublic(this.token, this.handle)).data;
        let title = this.userPublic!.full_name || this.userPublic!.handle;
        updateHead(this.$route.path, title, this.userPublic!.personal_introduction);

        const currentItem = this.tabItems.find((item) => item.code === this.currentTabItem);
        if (currentItem && currentItem.render) {
          await currentItem.render(this.userPublic);
        }
        await Promise.all(
          this.tabItems.map(async (item) => {
            if (item.code !== this.currentTabItem && this.userPublic) {
              if (item.render) {
                await item.render(this.userPublic);
              }
            }
          })
        );
      } else {
        this.userPublicForVisitor = (await apiPeople.getUserPublic(this.token, this.handle)).data;
      }
    });
  }

  private async loadAnswers(skip: number, limit: number) {
    let items: IAnswerPreview[] | null = null;
    await dispatchCaptureApiError(this.$store, async () => {
      if (this.userPublic !== null) {
        items = (await apiPeople.getAnswersByAuthor(this.token, this.userPublic.uuid, skip, limit))
          .data;
      }
    });
    return items;
  }

  private async loadQuestions(skip: number, limit: number) {
    let items: IQuestionPreview[] | null = null;
    await dispatchCaptureApiError(this.$store, async () => {
      if (this.userPublic !== null) {
        items = (
          await apiPeople.getQuestionsByAuthor(this.token, this.userPublic.uuid, skip, limit)
        ).data;
      }
    });
    return items;
  }

  private async loadSubmissions(skip: number, limit: number) {
    let items: ISubmission[] | null = null;
    await dispatchCaptureApiError(this.$store, async () => {
      if (this.userPublic !== null) {
        items = (
          await apiPeople.getSubmissionsByAuthor(this.token, this.userPublic.uuid, skip, limit)
        ).data;
      }
    });
    return items;
  }

  private async loadArticles(skip: number, limit: number) {
    let items: IArticlePreview[] | null = null;
    await dispatchCaptureApiError(this.$store, async () => {
      if (this.userPublic !== null) {
        items = (await apiPeople.getArticlesByAuthor(this.token, this.userPublic.uuid, skip, limit))
          .data;
      }
    });
    return items;
  }

  private async renderFollowers(userPublic: IUserPublic) {
    this.followers = (await apiPeople.getUserFollowers(this.token, userPublic.uuid)).data;
  }

  private async renderFollowed(userPublic: IUserPublic) {
    this.followed = (await apiPeople.getUserFollowed(this.token, userPublic.uuid)).data;
  }
}
</script>
