<template lang="pug">
  three
  .description
    h2 操作説明
    p WASDキーで車両を操作
    p Bキーで強制ブレーキ
    p Q、Eキーで車両を前から見る
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

<style lang="scss" scoped>
.description {
  position: fixed;
  top: 0;
  left: 0;
  z-index: 1;
  color: white;
}
</style>
