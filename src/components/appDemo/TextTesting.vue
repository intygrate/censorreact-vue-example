<template>
  <form class="cr-demo-text">

    <label class="cr-demo-text__label">
      Enter text:
      <div class="cr-demo-text__clear" :class="{show: clearText}" @click="deleteText()">clear X</div>
      <textarea
        rows="10"
        @keyup="checkLength"
        @paste="checkLength"
        v-model="text"
        class="cr-demo-text__textarea"
      />
    </label>

    <div class="cr-demo-text__stats">
      <p class="cr-demo-text__counter">{{ this.currentCount + '/' + this.maxCount }}</p>
      <button
        type="button"
        class="cr-demo-text__button"
        @click="runModeration()"
        :disabled="loading"
        :loading="loading"
      >Try Text API</button>
    </div>
  </form>
</template>

<script>
export default {
  name: 'TextTesting',
  data() {
    return {
      profile: 'default',
      loading: false,
      text: '',
      maxCount: 10000,
      currentCount: 0,
      url: `https://dev.api.censorreact.intygrate.com/devv1/text`,
      clearText: false,
    };
  },
  props: {
    token: {
      type: String,
    },
    showResultFn: {
      type: Function,
    },
  },
  methods: {
    deleteText() {
      this.text = '';
      this.currentCount = 0;
      this.clearText = false;
      this.showResultFn({ result: 'nothing to show..' });
    },
    async runModeration() {
      try {
        this.loading = true;
        this.showResultFn({ result: 'processing...' });

        const fetchRequest = await fetch(this.url, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'x-api-key': this.token,
          },
          body: JSON.stringify({ text: this.text, profile: this.profile }),
        });
        const data = await fetchRequest.json();

        this.showResultFn(data);
        this.loading = false;
      } catch (error) {
        this.loading = false;
        console.log('error', error);
      }
    },
    checkLength() {
      this.clearText = true;
      if (this.text.length < 1) {
        this.clearText = false;
      }
      if (this.text.length > 10000) {
        this.text = this.text.slice(0, 10000);
      }
      this.currentCount = this.text.length;
      this.showResultFn({ result: 'nothing to show..' });
      return this.text;
    },
  },
};
</script>
