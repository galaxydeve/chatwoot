<template>
  <div class="view-box fill-height">
    <banner
      v-if="!currentChat.can_reply"
      color-scheme="alert"
      :banner-message="replyWindowBannerMessage"
      :href-link="replyWindowLink"
      :href-link-text="replyWindowLinkText"
    />

    <banner
      v-if="isATweet"
      color-scheme="gray"
      :banner-message="tweetBannerText"
      :has-close-button="hasSelectedTweetId"
      @close="removeTweetSelection"
    />

    <div class="sidebar-toggle__wrap">
      <woot-button
        variant="smooth"
        size="tiny"
        color-scheme="secondary"
        class="sidebar-toggle--button"
        :icon="isRightOrLeftIcon"
        @click="onToggleContactPanel"
      />
    </div>
    <ul class="conversation-panel">
      <transition name="slide-up">
        <li class="spinner--container">
          <span v-if="shouldShowSpinner" class="spinner message" />
        </li>
      </transition>
      <message
        v-for="message in getReadMessages"
        :key="message.id"
        class="message--read ph-no-capture"
        data-clarity-mask="True"
        :data="message"
        :is-a-tweet="isATweet"
        :is-a-whatsapp-channel="isAWhatsAppChannel"
        :has-instagram-story="hasInstagramStory"
        :is-web-widget-inbox="isAWebWidgetInbox"
      />
      <li v-show="unreadMessageCount != 0" class="unread--toast">
        <span class="text-uppercase">
          {{ unreadMessageCount }}
          {{
            unreadMessageCount > 1
              ? $t('CONVERSATION.UNREAD_MESSAGES')
              : $t('CONVERSATION.UNREAD_MESSAGE')
          }}
        </span>
      </li>
      <message
        v-for="message in getUnReadMessages"
        :key="message.id"
        class="message--unread ph-no-capture"
        data-clarity-mask="True"
        :data="message"
        :is-a-tweet="isATweet"
        :is-a-whatsapp-channel="isAWhatsAppChannel"
        :has-instagram-story="hasInstagramStory"
        :is-web-widget-inbox="isAWebWidgetInbox"
      />
      <conversation-label-suggestion
        v-if="isEnterprise && isAIIntegrationEnabled"
        :suggested-labels="labelSuggestions"
        :chat-labels="currentChat.labels"
        :conversation-id="currentChat.id"
      />
    </ul>
    <div
      class="conversation-footer"
      :class="{ 'modal-mask': isPopoutReplyBox }"
    >
      <div v-if="isAnyoneTyping" class="typing-indicator-wrap">
        <div class="typing-indicator">
          {{ typingUserNames }}
          <img
            class="gif"
            src="~dashboard/assets/images/typing.gif"
            alt="Someone is typing"
          />
        </div>
      </div>
      <reply-box
        :conversation-id="currentChat.id"
        :is-a-tweet="isATweet"
        :selected-tweet="selectedTweet"
        :popout-reply-box.sync="isPopoutReplyBox"
        @click="showPopoutReplyBox"
      />
    </div>
  </div>
</template>

<script>
// components
import ReplyBox from './ReplyBox';
import Message from './Message';
import ConversationLabelSuggestion from './conversation/LabelSuggestion';
import Banner from 'dashboard/components/ui/Banner.vue';

// stores and apis
import { mapGetters } from 'vuex';

// mixins
import conversationMixin, {
  filterDuplicateSourceMessages,
} from '../../../mixins/conversations';
import inboxMixin from 'shared/mixins/inboxMixin';
import configMixin from 'shared/mixins/configMixin';
import eventListenerMixins from 'shared/mixins/eventListenerMixins';
import aiMixin from 'dashboard/mixins/aiMixin';

// utils
import { getTypingUsersText } from '../../../helper/commons';
import { calculateScrollTop } from './helpers/scrollTopCalculationHelper';
import { isEscape } from 'shared/helpers/KeyboardHelpers';

// constants
import { BUS_EVENTS } from 'shared/constants/busEvents';
import { REPLY_POLICY } from 'shared/constants/links';

export default {
  components: {
    Message,
    ReplyBox,
    Banner,
    ConversationLabelSuggestion,
  },
  mixins: [
    conversationMixin,
    inboxMixin,
    eventListenerMixins,
    configMixin,
    aiMixin,
  ],
  props: {
    isContactPanelOpen: {
      type: Boolean,
      default: false,
    },
  },

  data() {
    return {
      isLoadingPrevious: true,
      heightBeforeLoad: null,
      conversationPanel: null,
      selectedTweetId: null,
      hasUserScrolled: false,
      isProgrammaticScroll: false,
      isPopoutReplyBox: false,
      labelSuggestions: [],
    };
  },

  computed: {
    ...mapGetters({
      currentChat: 'getSelectedChat',
      allConversations: 'getAllConversations',
      inboxesList: 'inboxes/getInboxes',
      listLoadingStatus: 'getAllMessagesLoaded',
      loadingChatList: 'getChatListLoadingStatus',
      appIntegrations: 'integrations/getAppIntegrations',
      currentAccountId: 'getCurrentAccountId',
    }),
    inboxId() {
      return this.currentChat.inbox_id;
    },
    inbox() {
      return this.$store.getters['inboxes/getInbox'](this.inboxId);
    },
    hasSelectedTweetId() {
      return !!this.selectedTweetId;
    },
    tweetBannerText() {
      return !this.selectedTweetId
        ? this.$t('CONVERSATION.SELECT_A_TWEET_TO_REPLY')
        : `
          ${this.$t('CONVERSATION.REPLYING_TO')}
          ${this.selectedTweet.content}` || '';
    },
    typingUsersList() {
      const userList = this.$store.getters[
        'conversationTypingStatus/getUserList'
      ](this.currentChat.id);
      return userList;
    },
    isAnyoneTyping() {
      const userList = this.typingUsersList;
      return userList.length !== 0;
    },
    typingUserNames() {
      const userList = this.typingUsersList;

      if (this.isAnyoneTyping) {
        const userListAsName = getTypingUsersText(userList);
        return userListAsName;
      }

      return '';
    },
    getMessages() {
      const messages = this.currentChat.messages || [];
      if (this.isAWhatsAppChannel) {
        return filterDuplicateSourceMessages(messages);
      }
      return messages;
    },
    getReadMessages() {
      return this.readMessages(
        this.getMessages,
        this.currentChat.agent_last_seen_at
      );
    },
    getUnReadMessages() {
      return this.unReadMessages(
        this.getMessages,
        this.currentChat.agent_last_seen_at
      );
    },
    shouldShowSpinner() {
      return (
        (this.currentChat && this.currentChat.dataFetched === undefined) ||
        (!this.listLoadingStatus && this.isLoadingPrevious)
      );
    },
    conversationType() {
      const { additional_attributes: additionalAttributes } = this.currentChat;
      const type = additionalAttributes ? additionalAttributes.type : '';
      return type || '';
    },

    isATweet() {
      return this.conversationType === 'tweet';
    },

    hasInstagramStory() {
      return this.conversationType === 'instagram_direct_message';
    },

    selectedTweet() {
      if (this.selectedTweetId) {
        const { messages = [] } = this.currentChat;
        const [selectedMessage] = messages.filter(
          message => message.id === this.selectedTweetId
        );
        return selectedMessage || {};
      }
      return '';
    },
    isRightOrLeftIcon() {
      if (this.isContactPanelOpen) {
        return 'arrow-chevron-right';
      }
      return 'arrow-chevron-left';
    },
    getLastSeenAt() {
      const { contact_last_seen_at: contactLastSeenAt } = this.currentChat;
      return contactLastSeenAt;
    },

    replyWindowBannerMessage() {
      if (this.isAWhatsAppChannel) {
        return this.$t('CONVERSATION.TWILIO_WHATSAPP_CAN_REPLY');
      }
      if (this.isAPIInbox) {
        const { additional_attributes: additionalAttributes = {} } = this.inbox;
        if (additionalAttributes) {
          const {
            agent_reply_time_window_message: agentReplyTimeWindowMessage,
          } = additionalAttributes;
          return agentReplyTimeWindowMessage;
        }
        return '';
      }
      return this.$t('CONVERSATION.CANNOT_REPLY');
    },
    replyWindowLink() {
      if (this.isAWhatsAppChannel) {
        return REPLY_POLICY.FACEBOOK;
      }
      if (!this.isAPIInbox) {
        return REPLY_POLICY.TWILIO_WHATSAPP;
      }
      return '';
    },
    replyWindowLinkText() {
      if (this.isAWhatsAppChannel) {
        return this.$t('CONVERSATION.24_HOURS_WINDOW');
      }
      if (!this.isAPIInbox) {
        return this.$t('CONVERSATION.TWILIO_WHATSAPP_24_HOURS_WINDOW');
      }
      return '';
    },
    unreadMessageCount() {
      return this.currentChat.unread_count;
    },
  },

  watch: {
    currentChat(newChat, oldChat) {
      if (newChat.id === oldChat.id) {
        return;
      }
      this.fetchAllAttachmentsFromCurrentChat();
      this.fetchSuggestions();
      this.selectedTweetId = null;
    },
  },

  created() {
    bus.$on(BUS_EVENTS.SCROLL_TO_MESSAGE, this.onScrollToMessage);
    // when a new message comes in, we refetch the label suggestions
    bus.$on(BUS_EVENTS.FETCH_LABEL_SUGGESTIONS, this.fetchSuggestions);
    bus.$on(BUS_EVENTS.SET_TWEET_REPLY, this.setSelectedTweet);
  },

  mounted() {
    this.addScrollListener();
    this.fetchAllAttachmentsFromCurrentChat();
    this.fetchSuggestions();
  },

  beforeDestroy() {
    this.removeBusListeners();
    this.removeScrollListener();
  },

  methods: {
    async fetchSuggestions() {
      // start empty, this ensures that the label suggestions are not shown
      this.labelSuggestions = [];

      if (this.isLabelSuggestionDismissed()) {
        return;
      }

      if (!this.isEnterprise) {
        return;
      }

      // method available in mixin, need to ensure that integrations are present
      await this.fetchIntegrationsIfRequired();

      if (!this.isAIIntegrationEnabled) {
        return;
      }

      this.labelSuggestions = await this.fetchLabelSuggestions({
        conversationId: this.currentChat.id,
      });

      // once the labels are fetched, we need to scroll to bottom
      // but we need to wait for the DOM to be updated
      // so we use the nextTick method
      this.$nextTick(() => {
        // this param is added to route, telling the UI to navigate to the message
        // it is triggered by the SCROLL_TO_MESSAGE method
        // see setActiveChat on ConversationView.vue for more info
        const { messageId } = this.$route.query;

        // only trigger the scroll to bottom if the user has not scrolled
        // and there's no active messageId that is selected in view
        if (!messageId && !this.hasUserScrolled) {
          this.scrollToBottom();
        }
      });
    },
    isLabelSuggestionDismissed() {
      const dismissed = this.getDismissedConversations(this.currentAccountId);
      return dismissed[this.currentAccountId].includes(this.conversationId);
    },
    fetchAllAttachmentsFromCurrentChat() {
      this.$store.dispatch('fetchAllAttachments', this.currentChat.id);
    },
    removeBusListeners() {
      bus.$off(BUS_EVENTS.SCROLL_TO_MESSAGE, this.onScrollToMessage);
      bus.$off(BUS_EVENTS.SET_TWEET_REPLY, this.setSelectedTweet);
    },
    setSelectedTweet(tweetId) {
      this.selectedTweetId = tweetId;
    },
    onScrollToMessage({ messageId = '' } = {}) {
      this.$nextTick(() => {
        const messageElement = document.getElementById('message' + messageId);
        if (messageElement) {
          this.isProgrammaticScroll = true;
          messageElement.scrollIntoView({ behavior: 'smooth' });
          this.fetchPreviousMessages();
        } else {
          this.scrollToBottom();
        }
      });
      this.makeMessagesRead();
    },
    showPopoutReplyBox() {
      this.isPopoutReplyBox = !this.isPopoutReplyBox;
    },
    closePopoutReplyBox() {
      this.isPopoutReplyBox = false;
    },
    handleKeyEvents(e) {
      if (isEscape(e)) {
        this.closePopoutReplyBox();
      }
    },
    addScrollListener() {
      this.conversationPanel = this.$el.querySelector('.conversation-panel');
      this.setScrollParams();
      this.conversationPanel.addEventListener('scroll', this.handleScroll);
      this.$nextTick(() => this.scrollToBottom());
      this.isLoadingPrevious = false;
    },
    removeScrollListener() {
      this.conversationPanel.removeEventListener('scroll', this.handleScroll);
    },
    scrollToBottom() {
      this.isProgrammaticScroll = true;
      let relevantMessages = [];

      // label suggestions are not part of the messages list
      // so we need to handle them separately
      let labelSuggestions = this.conversationPanel.querySelector(
        '.label-suggestion'
      );

      // if there are unread messages, scroll to the first unread message
      if (this.unreadMessageCount > 0) {
        // capturing only the unread messages
        relevantMessages = this.conversationPanel.querySelectorAll(
          '.message--unread'
        );
      } else if (labelSuggestions) {
        // when scrolling to the bottom, the label suggestions is below the last message
        // so we scroll there if there are no unread messages
        // Unread messages always take the highest priority
        relevantMessages = [labelSuggestions];
      } else {
        // if there are no unread messages or label suggestion, scroll to the last message
        // capturing last message from the messages list
        relevantMessages = Array.from(
          this.conversationPanel.querySelectorAll('.message--read')
        ).slice(-1);
      }

      this.conversationPanel.scrollTop = calculateScrollTop(
        this.conversationPanel.scrollHeight,
        this.$el.scrollHeight,
        relevantMessages
      );
    },
    onToggleContactPanel() {
      this.$emit('contact-panel-toggle');
    },
    setScrollParams() {
      this.heightBeforeLoad = this.conversationPanel.scrollHeight;
      this.scrollTopBeforeLoad = this.conversationPanel.scrollTop;
    },

    async fetchPreviousMessages(scrollTop = 0) {
      this.setScrollParams();
      const shouldLoadMoreMessages =
        this.currentChat.dataFetched === true &&
        !this.listLoadingStatus &&
        !this.isLoadingPrevious;

      if (
        scrollTop < 100 &&
        !this.isLoadingPrevious &&
        shouldLoadMoreMessages
      ) {
        this.isLoadingPrevious = true;
        try {
          await this.$store.dispatch('fetchPreviousMessages', {
            conversationId: this.currentChat.id,
            before: this.currentChat.messages[0].id,
          });
          const heightDifference =
            this.conversationPanel.scrollHeight - this.heightBeforeLoad;
          this.conversationPanel.scrollTop =
            this.scrollTopBeforeLoad + heightDifference;
          this.setScrollParams();
        } catch (error) {
          // Ignore Error
        } finally {
          this.isLoadingPrevious = false;
        }
      }
    },

    handleScroll(e) {
      if (this.isProgrammaticScroll) {
        // Reset the flag
        this.isProgrammaticScroll = false;
        this.hasUserScrolled = false;
      } else {
        this.hasUserScrolled = true;
      }
      bus.$emit(BUS_EVENTS.ON_MESSAGE_LIST_SCROLL);
      this.fetchPreviousMessages(e.target.scrollTop);
    },

    makeMessagesRead() {
      this.$store.dispatch('markMessagesRead', { id: this.currentChat.id });
    },
    removeTweetSelection() {
      this.selectedTweetId = null;
    },
  },
};
</script>

<style scoped lang="scss">
.spinner--container {
  min-height: var(--space-jumbo);
}

.view-box.fill-height {
  height: auto;
  flex-grow: 1;
  min-width: 0;
}

.modal-mask {
  &::v-deep {
    .ProseMirror-woot-style {
      max-height: 25rem;
    }

    .reply-box {
      border: 1px solid var(--color-border);
      max-width: 75rem;
      width: 70%;
    }

    .reply-box .reply-box__top {
      position: relative;
      min-height: 27.5rem;
    }

    .reply-box__top .input {
      min-height: 27.5rem;
    }

    .emoji-dialog {
      position: fixed;
      left: unset;
      position: absolute;
      bottom: var(--space-smaller);
    }
  }
}
.sidebar-toggle__wrap {
  display: flex;
  justify-content: flex-end;

  .sidebar-toggle--button {
    position: fixed;

    top: var(--space-mega);
    z-index: var(--z-index-low);

    background: var(--white);

    padding: inherit 0;
    border-top-left-radius: calc(
      var(--space-medium) + 1px
    ); /* 100px of height + 10px of border */
    border-bottom-left-radius: calc(
      var(--space-medium) + 1px
    ); /* 100px of height + 10px of border */
    border: 1px solid var(--color-border-light);
    border-right: 0;
    box-sizing: border-box;
  }
}
</style>
