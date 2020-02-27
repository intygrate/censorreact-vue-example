<template>
  <section id="censorreact-demo-outer" class="censorreact-demo-outer">
    <div id="censorreact-demo" class="censorreact-demo">
      <h1 class="heading__primary heading--white">CensorREACT VUE Demo</h1>
      <div class="censorreact-demo__panel">
        <div class="tab-panel">
          <div class="tab-panel__tab" :class="{active: textMod}" @click="selectTextMod">
            <svg-text-icon class="tab-panel__icon" />
            <p class="heading__secondary heading--white tab-panel__title">Text</p>
          </div>
          <div class="tab-panel__tab tab-panel--img" :class="{active: imgMod}" @click="selectImgMod">
            <svg-image-icon class="tab-panel__icon" />
            <p class="heading__secondary heading--white tab-panel__title">Image</p>
          </div>
        </div>
        <div class="user-input">
          <div class="user-input__inner">
            <TextTesting v-if="textMod" :showResultFn="showResult" :token="token" />
            <ImageTesting v-if="imgMod" :showResultFn="showResult" :token="token" />
          </div>
        </div>
        <div class="demo-results">
          <div class="demo-results__inner">
            <pre class="demo-results__result-msg">{{ result }}</pre>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<script>
import SvgTextIcon from '@/components/icons/SvgTextIcon';
import SvgImageIcon from '@/components/icons/SvgImageIcon';
import TextTesting from '@/components/appDemo/TextTesting';
import ImageTesting from '@/components/appDemo/ImageTesting';

export default {
  name: 'AppDemo',
  components: {
    SvgTextIcon,
    SvgImageIcon,
    TextTesting,
    ImageTesting,
  },
  data() {
    return {
      textMod: true,
      imgMod: false,
      token: 'uRuFUQEnDv1PkRLCVzWPW84MMDZb79zL6p5QUZEz',
      result: { result: 'nothing to show..' },
    };
  },
  methods: {
    showResult(result) {
      this.result = result;
    },
    selectTextMod() {
      if (!this.textMod) {
        this.textMod = true;
        this.imgMod = false;
      }
    },
    selectImgMod() {
      if (!this.imgMod) {
        this.imgMod = true;
        this.textMod = false;
      }
    },
  },
};
</script>
