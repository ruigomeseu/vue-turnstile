<template>
  <div ref="turnstile"></div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import type { PropType } from 'vue';

const turnstileSrc = 'https://challenges.cloudflare.com/turnstile/v0/api.js';
const turnstileLoadFunction = 'cfTurnstileOnLoad';

declare global {
  interface Window {
    turnstile: any;
    [turnstileLoadFunction]: any;
  }
}

declare interface VueTurnstileData {
  resetTimeout?: ReturnType<typeof setTimeout>;
  widgetId?: string;
}

let turnstileState = typeof window !== 'undefined' ? (window.turnstile !== undefined ? 'ready' : 'unloaded') : 'unloaded';
let turnstileLoad: {
  resolve: (value?: unknown) => void;
  reject: (value?: unknown) => void;
};

export default defineComponent({
  name: 'VueTurnstile',

  emits: ['update:modelValue'],

  props: {
    siteKey: {
      type: String,
      required: true,
    },
    modelValue: {
      type: String,
      required: true,
    },
    resetInterval: {
      type: Number,
      required: false,
      default: 295 * 1000,
    },
    size: {
      type: String as PropType<'normal' | 'compact'>,
      required: false,
      default: 'normal',
    },
    theme: {
      type: String as PropType<'light' | 'dark' | 'auto'>,
      required: false,
      default: 'auto',
    },
    action: {
      type: String,
      required: false,
      default: '',
    },
    appearance: {
      type: String as PropType<'always' | 'execute' | 'interaction-only'>,
      required: false,
      default: 'always',
    },
    renderOnMount: {
      type: Boolean,
      required: false,
      default: true,
    },
  },

  data(): VueTurnstileData {
    return {
      resetTimeout: undefined,
      widgetId: undefined,
    };
  },

  computed: {
    turnstileOptions() {
      return {
        sitekey: this.siteKey,
        theme: this.theme,
        size: this.size,
        callback: this.callback,
        action: this.action,
        appearance: this.appearance,
      };
    },
  },

  methods: {
    callback(token: string) {
      this.$emit('update:modelValue', token);
      this.startResetTimeout();
    },

    reset() {
      if (window.turnstile) {
        this.$emit('update:modelValue', '');
        window.turnstile.reset();
      }
    },
    
    remove() {
      if (this.widgetId) {
        window.turnstile.remove(this.widgetId);
        this.widgetId = undefined;
      }
    },

    render() {
      this.widgetId = window.turnstile.render(this.$refs.turnstile, this.turnstileOptions);
    },

    startResetTimeout() {
      this.resetTimeout = setTimeout(() => {
        this.reset();
      }, this.resetInterval);
    },
  },

  async mounted() {
    const turnstileLoadPromise = new Promise((resolve, reject) => {
      turnstileLoad = { resolve, reject };
      if (turnstileState === 'ready') resolve(undefined);
    });

    window[turnstileLoadFunction] = () => {
      turnstileLoad.resolve();
      turnstileState = 'ready';
    };

    const ensureTurnstile = () => {
      if (turnstileState === 'unloaded') {
        turnstileState = 'loading';
        const url = `${turnstileSrc}?onload=${turnstileLoadFunction}&render=explicit`;
        const script = document.createElement('script');
        script.src = url;
        script.async = true;
        script.addEventListener('error', () => {
          turnstileLoad.reject('Failed to load Turnstile.');
        });
        document.head.appendChild(script);
      }
      return turnstileLoadPromise;
    };

    await ensureTurnstile();

    if (this.renderOnMount) {
      this.render();
    }
  },
  
  beforeUnmount() {
    this.remove();
    clearTimeout(this.resetTimeout);
  }
});
</script>
