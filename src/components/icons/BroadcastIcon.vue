<template>
  <v-icon v-bind="$attrs" v-on="$listeners">{{ svgPath }}</v-icon>
</template>

<script>
import { mdiBroadcast } from '@mdi/js';

export default {
  data: () => ({
    svgPath: mdiBroadcast,
  }),
};
</script>
