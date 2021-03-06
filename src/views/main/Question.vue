<template>
  <v-container fluid>
    <v-progress-linear v-if="question === null" indeterminate />
    <v-row v-else class="px-2" justify="center">
      <v-col
        :class="{
          'col-8': $vuetify.breakpoint.mdAndUp,
          'fixed-narrow-col': isNarrowFeedUI,
          'less-left-right-padding': !$vuetify.breakpoint.mdAndUp,
        }"
        class="mb-12"
        fluid
      >
        <!-- Question info/editor -->
        <div>
          <!-- Question topics display/editor -->
          <v-slide-group
            class="d-flex justify-space-between align-center"
            v-if="!showQuestionEditor"
          >
            <SiteBtn :site="question.site" class="elevation-0" />
            <v-chip-group>
              <v-chip
                small
                v-for="topic in question.topics"
                :key="topic.uuid"
                :to="'/topics/' + topic.uuid"
              >
                {{ topic.name }}
              </v-chip>
            </v-chip-group>
          </v-slide-group>
          <v-combobox
            v-else
            v-model="newQuestionTopicNames"
            :delimiters="[',', '，', '、']"
            :items="hintTopicNames"
            :label="$t('Topics')"
            hide-selected
            multiple
            small-chips
          />

          <!-- Question title display/editor -->
          <div class="pt-1">
            <div v-if="!showQuestionEditor" class="text-h5">
              {{ question.title }}
            </div>
            <v-textarea v-else v-model="newQuestionTitle" label="标题" auto-grow dense rows="1" />
          </div>

          <!-- Question description display/editor -->
          <Viewer
            v-if="!showQuestionEditor && question.description"
            :body="question.description"
            :editor="question.description_editor"
          />
          <div v-else-if="showQuestionEditor">
            <SimpleEditor
              ref="descEditor"
              :initialValue="question.description"
              :editor-prop="question.description_editor"
              placeholder="描述（选填）"
              :show-menu="true"
              class="mb-2"
            />
          </div>
        </div>

        <!-- Question control -->
        <v-slide-group v-if="!showQuestionEditor" class="d-flex my-1">
          <v-btn
            v-if="currentUserAnswerUUID"
            class="mr-1 slim-btn"
            depressed
            small
            @click="goToCurrentUserAnswer"
          >
            我的回答
          </v-btn>

          <v-btn
            v-else-if="savedLocalEdit"
            class="mr-1 slim-btn"
            color="primary"
            depressed
            small
            @click="loadSavedLocalEdit"
          >
            载入本地草稿
          </v-btn>

          <v-btn
            v-else-if="answerWritable"
            class="mr-1 slim-btn"
            color="primary"
            depressed
            small
            @click="showEditor = !showEditor"
          >
            写回答
          </v-btn>

          <v-tooltip v-else bottom>
            <template v-slot:activator="{ on, attrs }">
              <v-btn
                v-bind="attrs"
                v-on="on"
                :ripple="false"
                class="mr-1 slim-btn grey--text"
                depressed
                elevation="0"
                plain
                small
              >
                写回答
              </v-btn>
            </template>
            <span>该圈子仅会员可以写回答</span>
          </v-tooltip>

          <CommentBtn class="mr-1" @click="toggleShowComments" :count="question.comments.length" />

          <Upvote
            class="mr-1"
            v-if="upvotes !== null"
            :upvotes-count="upvotes.count"
            :upvoted="upvotes.upvoted"
            :disabled="userProfile.uuid === question.author.uuid"
            :on-cancel-vote="cancelUpvote"
            :on-vote="upvote"
          />

          <ShareCardButton
            class="mr-1"
            :link-text="question.title"
            :link="`/questions/${question.uuid}`"
            v-slot="{ shareQrCodeUrl }"
          >
            <v-card-title>
              {{ question.title }}
            </v-card-title>
            <v-card-text>
              <div class="pt-2 d-flex">
                <div class="text--primary text-body-1">
                  <p v-if="question.description" style="overflow-wrap: anywhere">
                    {{ question.description_text }}
                  </p>
                  <p>
                    <CommentsIcon class="mr-1" small />
                    <span class="text-caption"> {{ question.comments.length }}条评论 </span>
                  </p>
                  <p>
                    <AnswerIcon class="mr-1" small />
                    <span class="text-caption"> {{ question.answers_count }}个回答 </span>
                  </p>
                </div>
                <v-spacer />
                <div class="pa-1 text-center" style="float: right">
                  <v-img :src="shareQrCodeUrl" v-if="shareQrCodeUrl" max-width="100" />
                  <span class="text-caption">查看原问题</span>
                </div>
              </div>
            </v-card-text>
          </ShareCardButton>

          <v-menu offset-y>
            <template v-slot:activator="{ on, attrs }">
              <v-btn v-bind="attrs" v-on="on" class="slim-btn" depressed small>
                <DotsIcon small />
                <span class="ml-1" v-if="$vuetify.breakpoint.mdAndUp">更多</span>
              </v-btn>
            </template>
            <v-list dense>
              <v-list-item v-if="editable" @click="showQuestionEditor = true" dense>
                <EditIcon class="mr-1" />
                编辑
              </v-list-item>
              <v-list-item @click="showHistoryDialog" dense>
                <HistoryIcon v-if="editable && userProfile" class="mr-1" />
                问题历史
              </v-list-item>
              <v-list-item
                v-if="questionSubscription && questionSubscription.subscribed_by_me"
                :disabled="cancelSubscriptionIntermediate"
                @click="cancelSubscription"
                dense
              >
                <ToBookmarkIcon class="mr-1" />
                收藏
              </v-list-item>
              <v-list-item v-else :disabled="subscribeIntermediate" @click="subscribe" dense>
                <BookmarkedIcon class="mr-1" />
                取消收藏
              </v-list-item>
            </v-list>
          </v-menu>

          <v-dialog v-model="historyDialog" max-width="900">
            <v-card>
              <v-card-title primary-title>
                <div class="headline primary--text">问题历史</div>
                <v-spacer />
                <span class="text-caption grey--text">点击展开</span>
              </v-card-title>
              <v-expansion-panels>
                <v-expansion-panel v-for="archive in archives" :key="archive.id">
                  <v-expansion-panel-header>
                    {{ $dayjs.utc(archive.created_at).local().fromNow() }}
                    <span v-if="archive.editor"> - <UserLink :userPreview="archive.editor" /></span>
                    <v-spacer />
                  </v-expansion-panel-header>
                  <v-expansion-panel-content>
                    <v-chip-group>
                      <v-chip
                        v-for="topic in archive.topics"
                        :key="topic.uuid"
                        :to="'/topics/' + topic.uuid"
                        >{{ topic.name }}
                      </v-chip>
                    </v-chip-group>
                    <div class="headline primary--text">
                      {{ archive.title }}
                    </div>
                    <Viewer
                      v-if="archive.description_editor"
                      :body="archive.description"
                      :editor="archive.description_editor"
                    />
                  </v-expansion-panel-content>
                </v-expansion-panel>
              </v-expansion-panels>
            </v-card>
          </v-dialog>
        </v-slide-group>

        <!-- Question editor control -->
        <div v-if="showQuestionEditor && userProfile" class="d-flex">
          <v-btn
            v-show="editable"
            :disabled="commitQuestionEditIntermediate"
            class="ma-1 slim-btn"
            color="primary"
            depressed
            small
            @click="commitQuestionEdit"
            >保存
          </v-btn>
          <v-btn
            v-show="editable"
            class="ma-1 slim-btn"
            depressed
            small
            @click="cancelQuestionUpdate"
            >取消
          </v-btn>
          <v-spacer />
          <v-btn v-if="showQuestionEditor" depressed small @click="prepareShowMoveQuestionDialog">
            转移问题
          </v-btn>
          <v-dialog v-model="showMoveQuestionDialog" max-width="600">
            <v-card>
              <v-card-title primary-title>
                <div class="headline primary--text">转移问题</div>
              </v-card-title>
              <v-card-text>
                <span>现问题所属圈子: {{ question.site.name }}</span>
                <v-select
                  v-model="newQuestionSite"
                  :items="siteProfiles"
                  label="新的圈子"
                  item-text="site.name"
                  item-value="site"
                />
              </v-card-text>
              <v-card-actions>
                <span>没有权限？联系「Chafan猫管家」帮助转移</span>
                <v-spacer />
                <v-btn color="warning" depressed small @click="confirmMoveQuestion">
                  确认转移
                </v-btn>
              </v-card-actions>
            </v-card>
          </v-dialog>

          <v-btn
            v-if="showQuestionEditor & canHide"
            class="ml-2"
            color="warning"
            depressed
            small
            @click="showConfirmHideQuestionDialog = true"
          >
            隐藏问题
          </v-btn>
          <v-dialog v-model="showConfirmHideQuestionDialog" max-width="600">
            <v-card>
              <v-card-title primary-title>
                <div class="headline primary--text">确认隐藏问题</div>
              </v-card-title>
              <v-card-text>隐藏后问题将对所有用户不可见。</v-card-text>
              <v-card-actions>
                <v-spacer />
                <v-btn color="warning" depressed small @click="confirmHideQuestion"
                  >确认隐藏
                </v-btn>
              </v-card-actions>
            </v-card>
          </v-dialog>
        </div>

        <div class="d-flex justify-end mb-2">
          <ReactionBlock :objectId="question.uuid" objectType="question" />
        </div>

        <!-- Comments -->
        <v-expand-transition>
          <CommentBlock
            v-show="showComments"
            :commentSubmitIntermediate="commentSubmitIntermediate"
            :comments="question.comments"
            :siteId="question.site ? question.site.uuid : undefined"
            :writable="commentWritable"
            commentLabel="评论问题"
            @submit-new-comment="submitNewQuestionCommentBody"
          />
        </v-expand-transition>

        <div class="ml-2 text-center">
          <span v-if="showEditor && userProfile" class="text-caption grey--text">编辑答案</span>
          <span
            v-else-if="loadedFullAnswers.length === 0 && answerPreviews.length === 0"
            class="text-caption grey--text"
            >暂无答案</span
          >
        </div>

        <!-- Answer editor -->
        <v-expand-transition v-if="userProfile">
          <AnswerEditor
            :questionIdProp="question.uuid"
            v-if="showEditor"
            :inPrivateSite="!question.site.public_readable"
            class="ma-1"
            @updated-answer="updatedAnswerCallback"
            @cancel-edit="cancelHandler"
            @delete-draft="deleteDraft"
          />
        </v-expand-transition>

        <!-- Answers -->
        <div v-for="answer in loadedFullAnswers" :key="answer.uuid" class="shadow-card mb-4">
          <Answer
            :answerPreview="answer"
            :answerProp="answer"
            :loadFull="true"
            :showAuthor="true"
            :showCommentId="answerCommentId"
            :showQuestionInCard="false"
            @delete-answer="deleteHandler"
          />
        </div>

        <div
          v-for="answerPreview in answerPreviews"
          :key="answerPreview.uuid"
          class="shadow-card mb-4"
        >
          <Answer
            :answerPreview="answerPreview"
            :loadFull="answerPreviews.indexOf(answerPreview) <= 2"
            :showAuthor="true"
            :showCommentId="answerPreview.uuid === answerUUID ? answerCommentId : undefined"
            :showQuestionInCard="false"
            @load="loadingFullAnswer = false"
            @delete-answer="deleteHandler"
          />
        </div>

        <div v-if="answerPreviews.length && loadingFullAnswer" class="text-center">
          <v-progress-circular indeterminate size="20" color="grey" />
        </div>
      </v-col>

      <v-col
        v-if="$vuetify.breakpoint.mdAndUp"
        :class="isNarrowFeedUI ? 'fixed-narrow-sidecol' : 'col-4'"
      >
        <QuestionInfo :question="question" :questionSubscription="questionSubscription" />
      </v-col>
      <v-bottom-sheet v-else>
        <template v-slot:activator="{ on, attrs }">
          <v-btn v-bind="attrs" v-on="on" bottom fab fixed right>
            <InfoIcon />
          </v-btn>
        </template>
        <v-sheet class="pa-2">
          <QuestionInfo :question="question" :questionSubscription="questionSubscription" />
        </v-sheet>
      </v-bottom-sheet>
    </v-row>
  </v-container>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';

import Answer from '@/components/Answer.vue';
import QuestionInfo from '@/components/question/QuestionInfo.vue';
import SiteBtn from '@/components/SiteBtn.vue';
import ReactionBlock from '@/components/ReactionBlock.vue';
import CommentBlock from '@/components/CommentBlock.vue';
import UserLink from '@/components/UserLink.vue';

import BookmarkedIcon from '@/components/icons/BookmarkedIcon.vue';
import ToBookmarkIcon from '@/components/icons/ToBookmarkIcon.vue';
import HistoryIcon from '@/components/icons/HistoryIcon.vue';
import InfoIcon from '@/components/icons/InfoIcon.vue';
import EditIcon from '@/components/icons/EditIcon.vue';
import CommentsIcon from '@/components/icons/CommentsIcon.vue';

import {
  commitAddNotification,
  commitSetShowLoginPrompt,
  commitSetWorkingDraft,
} from '@/store/main/mutations';
import { api } from '@/api';

import {
  readIsLoggedIn,
  readNarrowUI,
  readToken,
  readUserMode,
  readUserProfile,
} from '@/store/main/getters';
import {
  IAnswer,
  IAnswerPreview,
  IQuestion,
  IQuestionArchive,
  IQuestionUpvotes,
  IRichEditorState,
  ISite,
  IUserQuestionSubscription,
  IUserSiteProfile,
} from '@/interfaces';
import Viewer from '@/components/Viewer.vue';
import SimpleEditor from '@/components/SimpleEditor.vue';
import {
  dispatchCaptureApiError,
  dispatchCaptureApiErrorWithErrorHandler,
} from '@/store/main/actions';
import { apiQuestion } from '@/api/question';
import { apiMe } from '@/api/me';
import { apiTopic } from '@/api/topic';
import { apiComment } from '@/api/comment';
import { AxiosError } from 'axios';
import { Route, RouteRecord } from 'vue-router';
import { isEqual, updateHead } from '@/common';
import { loadLocalEdit, LocalEdit } from '@/utils';
import AnswerIcon from '@/components/icons/AnswerIcon.vue';
import ShareCardButton from '@/components/ShareCardButton.vue';
import UpvoteBtn from '@/components/widgets/UpvoteBtn.vue';
import UpvotedBtn from '@/components/widgets/UpvotedBtn.vue';
import CommentBtn from '@/components/widgets/CommentBtn.vue';
import Upvote from '@/components/Upvote.vue';
import DotsIcon from '@/components/icons/DotsIcon.vue';

@Component({
  components: {
    DotsIcon,
    Upvote,
    CommentBtn,
    UpvotedBtn,
    UpvoteBtn,
    ShareCardButton,
    AnswerIcon,
    Answer,
    QuestionInfo,
    CommentBlock,
    UserLink,
    EditIcon,
    SiteBtn,
    ReactionBlock,
    BookmarkedIcon,
    ToBookmarkIcon,
    HistoryIcon,
    InfoIcon,
    CommentsIcon,
    Viewer,
    SimpleEditor,
  },
})
export default class Question extends Vue {
  private question: IQuestion | null = null;
  private showEditor: boolean = false;
  private newQuestionTitle: string = '';
  private showQuestionEditor: boolean = false;
  private showComments: boolean = false;
  private userSiteProfile: IUserSiteProfile | null = null;
  private questionSubscription: IUserQuestionSubscription | null = null;
  private newQuestionTopicNames: string[] = [];
  private hintTopicNames: string[] = []; // TODO
  private answerWritable = false;
  private commentWritable = false;
  private answerPreviews: IAnswerPreview[] = [];
  private loadedFullAnswers: IAnswer[] = [];
  private editable = false;
  private canHide = false;
  private showConfirmHideQuestionDialog = false;
  private loadingFullAnswer = true;
  private loading = true;
  private isModerator = false;
  private isShowInHome = false;
  private commitQuestionEditIntermediate = false;
  private cancelSubscriptionIntermediate = false;
  private subscribeIntermediate = false;
  private showMoveQuestionDialog = false;
  private savedNewAnswer: IAnswer | null = null;
  private handlingNewEdit = false;
  private upvotes: IQuestionUpvotes | null = null;
  private archives: IQuestionArchive[] = [];
  private historyDialog = false;
  private siteProfiles: IUserSiteProfile[] = [];
  private newQuestionSite: ISite | null = null;
  private currentUserAnswerUUID: string | null = null;
  private commentSubmitIntermediate = false;
  private savedLocalEdit: LocalEdit | null = null;

  get userProfile() {
    return readUserProfile(this.$store);
  }

  get isUserMode() {
    return readUserMode(this.$store);
  }

  get isNarrowFeedUI() {
    return readNarrowUI(this.$store);
  }

  get id() {
    return this.$route.params.id;
  }

  get answerUUID() {
    const aid = this.$route.params.aid;
    if (aid) {
      return aid;
    } else {
      return null;
    }
  }

  get answerCommentId() {
    const acid = this.$route.params.acid;
    if (acid) {
      return acid;
    } else {
      return null;
    }
  }

  get questionCommentId() {
    const qcid = this.$route.params.qcid;
    if (qcid) {
      return qcid;
    } else {
      return null;
    }
  }

  get token() {
    return readToken(this.$store);
  }

  private beforeRouteUpdate(to: Route, from: Route, next: () => void) {
    next();
    const matched = from.matched.find((record: RouteRecord) => record.name === 'question');
    if (matched && !isEqual(to.params, from.params)) {
      this.loading = true;
      this.showEditor = false;
      this.showQuestionEditor = false;
      this.showComments = false;
      this.answerWritable = false;
      this.commentWritable = false;
      this.answerPreviews = [];
      this.loadedFullAnswers = [];
      this.editable = false;
      this.canHide = false;
      this.showConfirmHideQuestionDialog = false;
      this.loadingFullAnswer = true;
      this.isModerator = false;
      this.isShowInHome = false;
      this.savedNewAnswer = null;
      this.upvotes = null;
      this.archives = [];
      this.siteProfiles = [];
      this.currentUserAnswerUUID = null;
      this.loadQuestion();
    }
  }

  private async loadQuestion() {
    await dispatchCaptureApiErrorWithErrorHandler(this.$store, {
      action: async () => {
        const response = await apiQuestion.getQuestion(this.token, this.id, true);
        this.question = response.data;
      },
      errorFilter: (err: AxiosError) => {
        if (
          err.response &&
          err.response.data.detail === "The question doesn't exists in the system."
        ) {
          commitAddNotification(this.$store, {
            content: '问题不存在，返回主页',
            color: 'error',
          });
          this.$router.push('/');
          return true;
        }
        return false;
      },
    });

    await dispatchCaptureApiError(this.$store, async () => {
      if (this.question) {
        if (readIsLoggedIn(this.$store)) {
          apiQuestion.bumpViewsCounter(this.token, this.question.uuid);
          this.upvotes = {
            question_uuid: this.question.uuid,
            count: this.question.upvotes_count,
            upvoted: this.question.upvoted,
          };

          if (this.questionCommentId !== null) {
            this.showComments = true;
          }

          this.isShowInHome = this.question.is_placed_at_home;

          updateHead(this.$route.path, this.question.title, this.question.description_text);

          this.newQuestionTitle = this.question.title;
          this.newQuestionTopicNames = this.question.topics.map((topic) => topic.name);
          if (this.userProfile) {
            if (this.userProfile.uuid === this.question.author.uuid) {
              this.editable = true;
              this.canHide = this.question.answers_count === 0;
            }
            const mod = this.question.site.moderator;
            if (mod) {
              this.isModerator = this.userProfile.uuid === mod.uuid;
            }
            if (this.isModerator) {
              this.canHide = this.question.answers_count === 0;
            }
          }
        }
        await dispatchCaptureApiError(this.$store, async () => {
          const answerPreviews = this.question?.answers
            ? this.question?.answers
            : (await api.getQuestionAnswers(this.token, this.question!.uuid)).data;
          if (this.answerUUID !== null) {
            if (!answerPreviews.find((preview) => preview.uuid === this.answerUUID)) {
              commitAddNotification(this.$store, {
                content: '答案不存在',
                color: 'error',
              });
            }
          }
          if (answerPreviews.length === 0 && !this.showQuestionEditor) {
            this.showEditor = true;
          }
          this.answerPreviews = answerPreviews.sort((a, b) => {
            if (a.uuid === this.answerUUID) {
              return -1;
            }
            if (b.uuid === this.answerUUID) {
              return 1;
            }
            if (a.upvotes_count > b.upvotes_count) {
              return -1;
            }
            return 1;
          });
          answerPreviews.forEach((a) => {
            if (a.author.uuid === this.userProfile?.uuid) {
              this.currentUserAnswerUUID = a.uuid;
            }
          });
          if (!this.currentUserAnswerUUID) {
            this.savedLocalEdit = loadLocalEdit('answer', 'answer-of-' + this.question!.uuid);
          }

          if (this.userProfile) {
            const questionSite = this.question!.site;

            this.userSiteProfile = (
              await api.getUserSiteProfile(this.token, questionSite.uuid, this.userProfile.uuid)
            ).data;
            if (this.userSiteProfile) {
              this.editable = true;
            }
            if (this.userSiteProfile !== null || questionSite.public_writable_answer) {
              this.answerWritable = true;
            }
            if (this.userSiteProfile !== null || questionSite.public_writable_comment) {
              this.commentWritable = true;
            }
            this.questionSubscription = (
              await apiMe.getQuestionSubscription(this.token, this.question!.uuid)
            ).data;
          }
        });
      }
    });
  }

  private async mounted() {
    try {
      if (localStorage.getItem('new-question')) {
        commitAddNotification(this.$store, {
          content: '点击「更多」编辑细节',
          color: 'info',
        });
        localStorage.removeItem('new-question');
      }
    } catch (e) {} // FIXME: is there a better way than just ignoring disabled localStorage?

    this.loadQuestion();
  }

  private updateOrAddFullyLoadedAnswer(answer: IAnswer) {
    const answerUUIDx = this.loadedFullAnswers.findIndex((a) => a.uuid === answer.uuid);
    if (answerUUIDx === -1) {
      this.loadedFullAnswers.unshift(answer);
    } else {
      this.loadedFullAnswers[answerUUIDx] = answer;
    }
  }

  private updatedAnswerCallback(event: { answer: IAnswer; isAutoSaved: boolean }) {
    this.handlingNewEdit = true;
    this.savedNewAnswer = event.answer;
    if (!event.isAutoSaved) {
      this.showEditor = false;
      this.updateOrAddFullyLoadedAnswer(event.answer);
    }
    this.handlingNewEdit = false;
  }

  private async submitNewQuestionCommentBody({ body, body_text, editor, mentioned }) {
    await dispatchCaptureApiError(this.$store, async () => {
      if (this.question) {
        this.commentSubmitIntermediate = true;
        const response = await apiComment.postComment(this.token, {
          site_uuid: this.question?.site.uuid,
          question_uuid: this.id,
          body,
          body_text,
          editor,
          mentioned,
        });
        const comment = response.data;
        this.question.comments.push(comment);
        this.commentSubmitIntermediate = false;
      }
    });
  }

  private async commitQuestionEdit() {
    this.commitQuestionEditIntermediate = true;
    await dispatchCaptureApiError(this.$store, async () => {
      const descEditor = this.$refs.descEditor as SimpleEditor;
      if (this.question && (this.newQuestionTitle || descEditor.content)) {
        const responses = await Promise.all(
          this.newQuestionTopicNames.map((name) => apiTopic.createTopic(this.token, { name }))
        );
        const topicsUUIDs = responses.map((r) => r.data.uuid);
        const response = await apiQuestion.updateQuestion(this.token, this.question.uuid, {
          title: this.newQuestionTitle,
          description: descEditor.content || undefined,
          description_text: descEditor.getTextContent() || undefined,
          description_editor: descEditor.editor,
          topic_uuids: topicsUUIDs,
        });
        if (response) {
          this.question = response.data;
        }
      }
      this.commitQuestionEditIntermediate = false;
      this.showQuestionEditor = false;
    });
  }

  private async cancelSubscription() {
    await dispatchCaptureApiError(this.$store, async () => {
      if (this.question) {
        this.cancelSubscriptionIntermediate = true;
        const r = await apiMe.unsubscribeQuestion(this.token, this.question.uuid);
        this.questionSubscription = r.data;
        this.cancelSubscriptionIntermediate = false;
      }
    });
  }

  private async subscribe() {
    if (!this.userProfile) {
      commitSetShowLoginPrompt(this.$store, true);
      return;
    }
    await dispatchCaptureApiError(this.$store, async () => {
      if (this.question) {
        this.subscribeIntermediate = true;
        const r = await apiMe.subscribeQuestion(this.token, this.question.uuid);
        this.questionSubscription = r.data;
        this.subscribeIntermediate = false;
      }
    });
  }

  private async upvote() {
    await dispatchCaptureApiError(this.$store, async () => {
      if (this.question) {
        this.upvotes = (await apiQuestion.upvoteQuestion(this.token, this.question.uuid)).data;
      }
    });
  }

  private async cancelUpvote() {
    await dispatchCaptureApiError(this.$store, async () => {
      if (this.question) {
        this.upvotes = (
          await apiQuestion.cancelUpvoteQuestion(this.token, this.question.uuid)
        ).data;
      }
    });
  }

  private deleteDraft() {
    this.savedLocalEdit = null;
    this.showEditor = false;
  }

  private async showHistoryDialog() {
    if (this.question) {
      this.archives = (await apiQuestion.getQuestionArchives(this.token, this.question.uuid)).data;
      this.archives.unshift({
        id: 0,
        title: this.question.title,
        description: this.question.description,
        description_text: this.question.description_text,
        description_editor: this.question.description_editor,
        topics: this.question.topics,
        created_at: this.question.updated_at,
        editor: this.question.editor,
      });
      if (this.archives.length > 0) {
        this.historyDialog = true;
      } else {
        commitAddNotification(this.$store, {
          content: '尚无历史存档',
          color: 'info',
        });
      }
    }
  }

  private cancelHandler() {
    this.showEditor = false;
    if (this.question && this.savedNewAnswer) {
      this.loadedFullAnswers.unshift(this.savedNewAnswer);
    }
  }

  private removeAnswer(answerUUID: string) {
    if (this.answerPreviews) {
      let idx = this.loadedFullAnswers.findIndex((answer) => answer.uuid === answerUUID);
      if (idx !== -1) {
        this.loadedFullAnswers.splice(idx, 1);
      }
      idx = this.answerPreviews.findIndex((answer) => answer.uuid === answerUUID);
      if (idx !== -1) {
        this.answerPreviews.splice(idx, 1);
      }
    }
  }

  private deleteHandler(answerUUID: string) {
    this.removeAnswer(answerUUID);
    if (this.answerUUID === answerUUID) {
      this.$router.push(`/questions/${this.question?.uuid}`);
    }
  }

  private async prepareShowMoveQuestionDialog() {
    this.siteProfiles = (await apiMe.getUserSiteProfiles(this.token)).data;
    this.showMoveQuestionDialog = true;
  }

  private async confirmMoveQuestion() {
    if (this.question) {
      if (!this.newQuestionSite) {
        commitAddNotification(this.$store, {
          content: this.$t('未选择新的圈子').toString(),
          color: 'info',
        });
      }
      if (this.question.site.uuid === this.newQuestionSite?.uuid) {
        commitAddNotification(this.$store, {
          content: this.$t('没有变化').toString(),
          color: 'info',
        });
      }
      await dispatchCaptureApiError(this.$store, async () => {
        this.question = (
          await apiQuestion.moveQuestion(
            this.$store.state.main.token,
            this.question!.uuid,
            this.newQuestionSite!.uuid
          )
        ).data;
        commitAddNotification(this.$store, {
          content: this.$t('转移成功').toString(),
          color: 'success',
        });
        this.showMoveQuestionDialog = false;
        this.showQuestionEditor = false;
      });
    }
  }

  private async confirmHideQuestion() {
    await dispatchCaptureApiError(this.$store, async () => {
      await apiQuestion.hideQuestion(this.$store.state.main.token, this.question!.uuid);
      commitAddNotification(this.$store, {
        content: this.$t('已隐藏').toString(),
        color: 'info',
      });
      this.$router.push(`/sites/${this.question!.site.subdomain}`);
    });
  }

  private async goToCurrentUserAnswer() {
    if (this.currentUserAnswerUUID) {
      this.$router.replace(
        `/questions/${this.question?.uuid}/answers/${this.currentUserAnswerUUID}`
      );
    }
  }

  private toggleShowComments() {
    if (!this.userProfile && this.question!.comments.length === 0) {
      commitSetShowLoginPrompt(this.$store, true);
      return;
    }
    this.showComments = !this.showComments;
  }

  private loadSavedLocalEdit() {
    commitSetWorkingDraft(this.$store, this.savedLocalEdit!.edit as IRichEditorState);
    this.showEditor = true;
  }

  private async cancelQuestionUpdate() {
    this.showQuestionEditor = false;
  }
}
</script>
