<template>
  <li v-if="shouldShowSuggestions" class="label-suggestion right">
    <div class="wrap">
      <div class="label-suggestion--container">
        <h6 class="label-suggestion--title">Suggested labels</h6>
        <div v-if="!fetchingSuggestions" class="label-suggestion--options">
          <button
            v-for="label in preparedLabels"
            :key="label.title"
            v-tooltip.top="{
              content: selectedLabels.includes(label.title)
                ? $t('LABEL_MGMT.SUGGESTIONS.TOOLTIP.DESELECT')
                : labelTooltip,
              delay: { show: 600, hide: 0 },
              hideOnClick: true,
            }"
            class="label-suggestion--option"
            @click="pushOrAddLabel(label.title)"
          >
            <woot-label
              variant="dashed"
              v-bind="label"
              :bg-color="
                selectedLabels.includes(label.title) ? 'var(--w-100)' : ''
              "
            />
          </button>
          <woot-button
            v-if="preparedLabels.length === 1"
            v-tooltip.top="{
              content: $t('LABEL_MGMT.SUGGESTIONS.TOOLTIP.DISMISS'),
              delay: { show: 600, hide: 0 },
              hideOnClick: true,
            }"
            variant="smooth"
            class="label--add"
            icon="dismiss"
            size="tiny"
            @click="dismissSuggestions"
          />
        </div>
        <div v-if="preparedLabels.length > 1">
          <woot-button
            variant="smooth"
            class="label--add"
            icon="add"
            size="tiny"
            @click="addAllLabels"
          >
            {{ addButtonText }}
          </woot-button>
          <woot-button
            v-tooltip.top="{
              content: $t('LABEL_MGMT.SUGGESTIONS.TOOLTIP.DISMISS'),
              delay: { show: 600, hide: 0 },
              hideOnClick: true,
            }"
            variant="smooth"
            class="label--add"
            icon="dismiss"
            size="tiny"
            @click="dismissSuggestions"
          />
        </div>
      </div>
      <div class="sender--info has-tooltip" data-original-title="null">
        <woot-thumbnail
          v-tooltip.top="{
            content: $t('LABEL_MGMT.SUGGESTIONS.POWERED_BY'),
            delay: { show: 600, hide: 0 },
            hideOnClick: true,
          }"
          size="16px"
        >
          <avatar class="user-thumbnail thumbnail-rounded">
            <fluent-icon class="chatwoot-ai-icon" icon="chatwoot-ai" />
          </avatar>
        </woot-thumbnail>
      </div>
    </div>
  </li>
</template>

<script>
// components
import WootButton from '../../../ui/WootButton.vue';
import Avatar from '../../Avatar.vue';
import aiMixin from 'dashboard/mixins/aiMixin';

// store & api
import { mapGetters } from 'vuex';

// utils & constants
import { LocalStorage } from 'shared/helpers/localStorage';
import { LOCAL_STORAGE_KEYS } from 'dashboard/constants/localStorage';
import { OPEN_AI_EVENTS } from '../../../../helper/AnalyticsHelper/events';

export default {
  name: 'LabelSuggestion',
  components: {
    Avatar,
    WootButton,
  },
  mixins: [aiMixin],
  props: {
    suggestedLabels: {
      type: Array,
      required: true,
    },
    chatLabels: {
      type: Array,
      required: false,
      default: () => [],
    },
    conversationId: {
      type: Number,
      required: true,
    },
  },
  data() {
    return {
      isDismissed: false,
      fetchingSuggestions: false,
      selectedLabels: [],
    };
  },
  computed: {
    ...mapGetters({
      allLabels: 'labels/getLabels',
      currentAccountId: 'getCurrentAccountId',
    }),
    labelTooltip() {
      if (this.preparedLabels.length > 1) {
        return this.$t('LABEL_MGMT.SUGGESTIONS.TOOLTIP.MULTIPLE_SUGGESTION');
      }

      return this.$t('LABEL_MGMT.SUGGESTIONS.TOOLTIP.SINGLE_SUGGESTION');
    },
    addButtonText() {
      if (this.selectedLabels.length === 1) {
        return this.$t('LABEL_MGMT.SUGGESTIONS.ADD_SELECTED_LABEL');
      }

      if (this.selectedLabels.length > 1) {
        return this.$t('LABEL_MGMT.SUGGESTIONS.ADD_SELECTED_LABELS');
      }

      return this.$t('LABEL_MGMT.SUGGESTIONS.ADD_ALL_LABELS');
    },
    preparedLabels() {
      return this.allLabels.filter(label =>
        this.suggestedLabels.includes(label.title)
      );
    },
    shouldShowSuggestions() {
      if (this.isDismissed) return false;
      if (!this.isAIIntegrationEnabled) return false;

      return (
        !this.fetchingSuggestions &&
        this.preparedLabels.length &&
        this.chatLabels.length === 0
      );
    },
  },
  watch: {
    conversationId: {
      immediate: true,
      handler() {
        this.selectedLabels = [];
        this.isDismissed = this.isConversationDismissed();
      },
    },
  },
  methods: {
    pushOrAddLabel(label) {
      if (this.preparedLabels.length === 1) {
        this.addAllLabels();
        return;
      }

      if (!this.selectedLabels.includes(label)) {
        this.selectedLabels.push(label);
      } else {
        this.selectedLabels = this.selectedLabels.filter(l => l !== label);
      }
    },
    dismissSuggestions() {
      const dismissed = this.getDismissedConversations(this.currentAccountId);
      dismissed[this.currentAccountId].push(this.conversationId);

      LocalStorage.set(
        LOCAL_STORAGE_KEYS.DISMISSED_LABEL_SUGGESTIONS,
        dismissed
      );

      // dismiss this once the values are set
      this.isDismissed = true;
      this.trackLabelEvent(OPEN_AI_EVENTS.DISMISS_LABEL_SUGGESTION);
    },
    isConversationDismissed() {
      const dismissed = this.getDismissedConversations(this.currentAccountId);
      return dismissed[this.currentAccountId].includes(this.conversationId);
    },
    addAllLabels() {
      let labelsToAdd = this.selectedLabels;
      if (!labelsToAdd.length) {
        labelsToAdd = this.preparedLabels.map(label => label.title);
      }
      this.$store.dispatch('conversationLabels/update', {
        conversationId: this.conversationId,
        labels: labelsToAdd,
      });
      this.trackLabelEvent(OPEN_AI_EVENTS.LABEL_SUGGESTION_APPLIED);
    },
    trackLabelEvent(event) {
      const payload = {
        conversationId: this.conversationId,
        account: this.currentAccountId,
        suggestions: this.suggestedLabels,
        labelsApplied: this.selectedLabels.length
          ? this.selectedLabels
          : this.suggestedLabels,
      };

      this.$track(event, payload);
    },
  },
};
</script>

<style scoped lang="scss">
.wrap {
  display: flex;
}

.label-suggestion {
  flex-direction: row;
  justify-content: flex-end;
  margin-top: var(--space-normal);

  .label-suggestion--container {
    max-width: 300px;
  }

  .label-suggestion--options {
    text-align: right;
    display: flex;
    align-items: center;
    gap: var(--space-micro);

    button.label-suggestion--option {
      .label {
        cursor: pointer;
        margin-bottom: 0;
      }
    }
  }

  .chatwoot-ai-icon {
    height: var(--font-size-mini);
    width: var(--font-size-mini);
  }

  .label-suggestion--title {
    color: var(--b-600);
    margin-top: var(--space-micro);
    font-size: var(--font-size-micro);
  }
}
</style>
