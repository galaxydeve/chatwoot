<template>
  <div v-if="isAIIntegrationEnabled" class="position-relative">
    <div v-if="!message">
      <woot-button
        v-if="isPrivateNote"
        v-tooltip.top-end="$t('INTEGRATION_SETTINGS.OPEN_AI.SUMMARY_TITLE')"
        icon="book-pulse"
        color-scheme="secondary"
        variant="smooth"
        size="small"
        :is-loading="uiFlags.summarize"
        @click="processEvent('summarize')"
      />
      <woot-button
        v-else
        v-tooltip.top-end="$t('INTEGRATION_SETTINGS.OPEN_AI.REPLY_TITLE')"
        icon="wand"
        color-scheme="secondary"
        variant="smooth"
        size="small"
        :is-loading="uiFlags.reply_suggestion"
        @click="processEvent('reply_suggestion')"
      />
    </div>

    <div v-else>
      <woot-button
        v-tooltip.top-end="$t('INTEGRATION_SETTINGS.OPEN_AI.TITLE')"
        icon="text-grammar-wand"
        color-scheme="secondary"
        variant="smooth"
        size="small"
        @click="toggleDropdown"
      />
      <div
        v-if="showDropdown"
        v-on-clickaway="closeDropdown"
        class="dropdown-pane dropdown-pane--open ai-modal"
      >
        <h4 class="sub-block-title margin-top-1">
          {{ $t('INTEGRATION_SETTINGS.OPEN_AI.TITLE') }}
        </h4>
        <p>
          {{ $t('INTEGRATION_SETTINGS.OPEN_AI.SUBTITLE') }}
        </p>
        <label>
          {{ $t('INTEGRATION_SETTINGS.OPEN_AI.TONE.TITLE') }}
        </label>
        <div class="tone__item">
          <select v-model="activeTone" class="status--filter small">
            <option v-for="tone in tones" :key="tone.key" :value="tone.key">
              {{ tone.value }}
            </option>
          </select>
        </div>
        <div class="modal-footer flex-container align-right">
          <woot-button variant="clear" size="small" @click="closeDropdown">
            {{ $t('INTEGRATION_SETTINGS.OPEN_AI.BUTTONS.CANCEL') }}
          </woot-button>
          <woot-button
            :is-loading="uiFlags.rephrase"
            size="small"
            @click="processEvent('rephrase')"
          >
            {{ buttonText }}
          </woot-button>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import { mixin as clickaway } from 'vue-clickaway';
import OpenAPI from 'dashboard/api/integrations/openapi';
import alertMixin from 'shared/mixins/alertMixin';
import aiMixin from 'dashboard/mixins/aiMixin';
import eventListenerMixins from 'shared/mixins/eventListenerMixins';
import { buildHotKeys } from 'shared/helpers/KeyboardHelpers';

export default {
  mixins: [aiMixin, alertMixin, clickaway, eventListenerMixins],
  props: {
    conversationId: {
      type: Number,
      default: 0,
    },
    message: {
      type: String,
      default: '',
    },
    isPrivateNote: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      uiFlags: {
        rephrase: false,
        reply_suggestion: false,
        summarize: false,
      },
      showDropdown: false,
      activeTone: 'professional',
      initialMessage: '',
      tones: [
        {
          key: 'professional',
          value: this.$t(
            'INTEGRATION_SETTINGS.OPEN_AI.TONE.OPTIONS.PROFESSIONAL'
          ),
        },
        {
          key: 'friendly',
          value: this.$t('INTEGRATION_SETTINGS.OPEN_AI.TONE.OPTIONS.FRIENDLY'),
        },
      ],
    };
  },
  computed: {
    buttonText() {
      return this.uiFlags.isRephrasing
        ? this.$t('INTEGRATION_SETTINGS.OPEN_AI.BUTTONS.GENERATING')
        : this.$t('INTEGRATION_SETTINGS.OPEN_AI.BUTTONS.GENERATE');
    },
  },
  methods: {
    onKeyDownHandler(event) {
      const keyPattern = buildHotKeys(event);
      const shouldRevertTheContent =
        ['meta+z', 'ctrl+z'].includes(keyPattern) && !!this.initialMessage;

      if (shouldRevertTheContent) {
        this.$emit('replace-text', this.initialMessage);
        this.initialMessage = '';
      }
    },
    toggleDropdown() {
      this.showDropdown = !this.showDropdown;
    },
    closeDropdown() {
      this.showDropdown = false;
    },
    async processEvent(type = 'rephrase') {
      this.uiFlags[type] = true;
      try {
        const result = await OpenAPI.processEvent({
          hookId: this.hookId,
          type,
          content: this.message,
          tone: this.activeTone,
          conversationId: this.conversationId,
        });
        const {
          data: { message: generatedMessage },
        } = result;
        this.initialMessage = this.message;
        this.$emit('replace-text', generatedMessage || this.message);
        this.closeDropdown();
        this.recordAnalytics(type, { tone: this.activeTone });
      } catch (error) {
        this.showAlert(this.$t('INTEGRATION_SETTINGS.OPEN_AI.GENERATE_ERROR'));
      } finally {
        this.uiFlags[type] = false;
      }
    },
  },
};
</script>

<style lang="scss" scoped>
.ai-modal {
  width: 400px;
  right: 0;
  left: 0;
  padding: var(--space-normal);
  bottom: 34px;
  position: absolute;
  span {
    font-size: var(--font-size-small);
    font-weight: var(--font-weight-medium);
  }

  p {
    color: var(--s-600);
  }

  label {
    margin-bottom: var(--space-smaller);
  }

  .status--filter {
    background-color: var(--color-background-light);
    border: 1px solid var(--color-border);
    font-size: var(--font-size-small);
    height: var(--space-large);
    padding: 0 var(--space-medium) 0 var(--space-small);
  }

  .modal-footer {
    gap: var(--space-smaller);
  }
}
</style>
