<template>
  <v-icon v-bind="$attrs" v-on="$listeners">{{ svgPath }}</v-icon>
</template>

<script>
import { mdiLinkedin } from '@mdi/js';

export default {
  data: () => ({
    svgPath: mdiLinkedin,
  }),
};
</script>
