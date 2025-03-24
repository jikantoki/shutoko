<template lang="pug">
  h1 Hello World!
</template>

<script>
/**
 * ページ推移ごとにmountedを実行する必要があるため、どのviewsでも読み込むこと
 */
import mixins from '~/mixins/mixins'
import webpush from '~/js/webpush'
import metaFunctions from '~/js/metaFunctions'
import Setup from '~/js/setup'
export default {
  name: 'index',
  components: {},
  mixins: [mixins],
  setup() {},
  data() {
    return {}
  },
  async mounted() {},
  methods: {},
}
</script>
