<template lang="pug">
.v-app-main-application#nuxt
  splash(v-show="splash")
  v-app
    nuxt-page
</template>

<script>
import PackageJson from '/package.json'
import Functions from '~/js/Functions'
import mixins from '~/mixins/mixins'
import webpush from '~/js/webpush'
import splash from '~/components/common/commonSplash'

export default {
  /**
   * Appが定番
   */
  name: 'App',
  /**
   * 使いたいvueファイルを記述
   */
  components: {
    splash: splash,
  },
  mixins: [mixins],
  /**
   * このファイル内で使うthisで始まる変数
   */
  data() {
    /**
     * この中に書かないと使えない
     */
    return {
      splash: true,
    }
  },
  /**
   * 監視したい変数を記述
   */
  watch: {},
  /**
   * ページ生成時にやりたい事
   */
  created() {},
  mounted() {
    /**
     * mountedの最後に記述
     */
    window.setTimeout(() => {
      this.splash = false
      this.style = 'opacity: 1;'
    }, 2000)
  },
  /**
   * ページ離脱時にやりたい事
   */
  unmouonted() {},
  /**
   * このファイル内で使いたい変数
   */
  methods: {},
}
</script>

<style lang="scss">
$breakpoints: (
  'smartPhone': 'screen and (max-width:700px)',
  'tablet': 'screen and (max-width:1100px)',
  'pwa': '(display-mode: standalone)',
) !default;

@mixin mq($breakpoint) {
  @media #{map-get($breakpoints, $breakpoint)} {
    @content;
  }
}

/* フォント設定 */
$font: 'Zen Maru Gothic', sans-serif;
@import url('https://fonts.googleapis.com/css2?family=Zen+Maru+Gothic&display=swap');
/* フォント設定ここまで */

$body-font-family: $font;
html,
body {
  height: 100vh !important;
}
:root {
  font-size: 16px;
  --color-allow: #cceeff;
  --color-error: #ffcccc;
  --color-error: #cc2222;
  --color-success: #338833;
  /** アプリの色 */
  --accent-color: #00bbee;
  /** アプリの色に合わせた文字色 */
  --accent-text-color: #ffffff;
  /** デフォルトのボーダー角の大きさ */
  --border-radius: 16px;
}
* {
  user-select: none;
  list-style: none;
  transition: all 0.14s;
  font-family: $font !important;
}
body {
  font-family: $font !important;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  .center {
    @include mq('tablet') {
      width: 80%;
    }
    @include mq('smartPhone') {
      width: 100%;
    }
  }
}
</style>
