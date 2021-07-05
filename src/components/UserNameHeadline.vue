<template>
  <div class="pt-6">
    <router-link :to="'/users/' + userPreview.handle" class="text-decoration-none headline">
      <span v-if="userPreview.full_name">
        {{ userPreview.full_name }}
      </span>
      <span v-else> @{{ userPreview.handle }} </span>
    </router-link>

    <span v-if="userPreview.full_name" class="grey--text headline">
      (@{{ userPreview.handle }})
    </span>

    <div v-if="userPreview.personal_introduction" class="secondary--text text-body-1">
      {{ userPreview.personal_introduction }}
    </div>
  </div>
</template>

<script lang="ts">
import { IUserPreview } from '@/interfaces';
import { Component, Vue, Prop } from 'vue-property-decorator';

@Component
export default class userNameHeadline extends Vue {
  @Prop() public readonly userPreview!: IUserPreview;
}
</script>
