<template>
  <v-icon v-bind="$attrs" v-on="$listeners">{{ svgPath }}</v-icon>
</template>

<script>
import { mdiLifebuoy } from '@mdi/js';

export default {
  data: () => ({
    svgPath: mdiLifebuoy,
  }),
};
</script>
