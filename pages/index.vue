<template lang="pug">
  three
</template>

<script>
/**
 * ページ推移ごとにmountedを実行する必要があるため、どのviewsでも読み込むこと
 */
import mixins from '~/mixins/mixins'
import webpush from '~/js/webpush'
import metaFunctions from '~/js/metaFunctions'
import Setup from '~/js/setup'
import three from '~/components/three.vue'
export default {
  name: 'index',
  components: {
    three,
  },
  mixins: [mixins],
  setup() {},
  data() {
    return {}
  },
  async mounted() {},
  methods: {},
}
</script>
