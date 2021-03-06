<template>
  <div v-if="comments.length > 0 || loggedIn" class="mt-3">
    <v-divider v-if="comments.length" />
    <span v-if="showTitle" class="pt-1 pl-1 grey--text">共{{ comments.length }}条评论</span>
    <v-list-item v-for="comment in comments" :key="comment.uuid" class="comment-item">
      <v-list-item-content>
        <Comment
          :comment="comment"
          :enableUpvotes="enableUpvotes"
          :siteId="siteId"
          :writable="writable"
        />
      </v-list-item-content>
    </v-list-item>
    <div v-if="writable" class="pb-1">
      <SimpleEditor
        ref="simpleEditor"
        :onMentionedHandles="onMentionedHandles"
        placeholder="评论"
        class="mb-1"
      />
      <div class="d-flex pt-1">
        <span v-if="mentioned.length" class="grey--text caption">
          将通知用户：<v-chip small v-for="handle in mentioned" :key="handle">{{ handle }}</v-chip>
        </span>
        <v-spacer />
        <v-btn
          :disabled="commentSubmitIntermediate"
          color="primary"
          depressed
          small
          @click="submitNewComment"
        >
          提交评论
          <v-progress-circular v-show="commentSubmitIntermediate" :size="20" indeterminate />
        </v-btn>
      </div>
    </div>
    <div v-else-if="loggedIn" class="text-caption grey--text pl-2">仅圈子成员可评论</div>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Prop, Emit } from 'vue-property-decorator';
import { commitAddNotification } from '@/store/main/mutations';
import UserLink from '@/components/UserLink.vue';
import Comment from '@/components/Comment.vue';
import SimpleEditor from '@/components/SimpleEditor.vue';
import { IComment } from '@/interfaces';
import { readIsLoggedIn } from '@/store/main/getters';

@Component({
  components: { UserLink, Comment, SimpleEditor },
})
export default class CommentBlock extends Vue {
  @Prop() public readonly comments!: IComment[];
  @Prop() public readonly writable!: boolean;
  @Prop() public readonly siteId: number | undefined;
  @Prop() public readonly commentLabel!: string;
  @Prop({ default: false }) public readonly showTitle!: boolean;
  @Prop({ default: false }) public commentSubmitIntermediate!: boolean;
  @Prop({ default: false }) public readonly enableUpvotes!: boolean;

  private mentioned: string[] = [];

  get loggedIn() {
    return readIsLoggedIn(this.$store);
  }

  @Emit('submit-new-comment')
  public submitNewComment() {
    const editor = this.$refs.simpleEditor as SimpleEditor;
    if (!editor.content || editor.content.length === 0) {
      commitAddNotification(this.$store, {
        content: this.$t("Comment can't be empty.").toString(),
        color: 'error',
      });
      return;
    }
    const commentCopy = `${editor.content}`;
    const commentText = editor.getTextContent();
    editor.reset();
    return {
      body: commentCopy,
      body_text: commentText,
      editor: editor.editor,
      mentioned: this.mentioned,
    };
  }

  private onMentionedHandles(handles: string[]) {
    this.mentioned = handles;
  }
}
</script>
