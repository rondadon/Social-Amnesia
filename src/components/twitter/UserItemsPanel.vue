<template>
  <div class="tweetsContainer" v-if="loggedIn">
    <h1>{{ this.itemtype === "tweets" ? "Your tweets" : "Your favorites" }}</h1>
    <b-pagination
      v-model="currentPage"
      :total-rows="rows"
      :per-page="perPage"
      aria-controls="itemList"
      align="center"
    />
    <ul id="itemList" class="tweetList">
      <li v-for="tweet in userItems" :key="`${itemtype}-${tweet.id}`">
        <div class="tweetAndOptionsContainer">
          <div class="tweetOptions">
            <div class="tweetWhitelist">
              <b-form-checkbox
                switch
                :id="`checklist-${itemtype}-${tweet.id}`"
                v-on:change="handleChanged(tweet)"
                :checked="checkIfSelected(tweet)"
              />
              <span>Whitelist</span>
            </div>
            <div class="tweetDeleteIconContainer">
              <b-icon
                v-b-tooltip.hover.bottom
                :title="
                  itemtype === 'tweets' ? 'Delete this tweet' : 'Remove this ❤️'
                "
                icon="trash"
                class="tweetDeleteIcon"
                v-on:click="handleDeleteItemClicked(tweet)"
              />
            </div>
          </div>
          <a
            class="tweet"
            :href="
              `https://twitter.com/${tweet.user.screen_name}/status/${tweet.id_str}`
            "
            target="_blank"
            @click.prevent="
              openExternalBrowser(
                `https://twitter.com/${tweet.user.screen_name}/status/${tweet.id_str}`
              )
            "
          >
            <div class="tweetHeader">
              <img :src="tweet.user.profile_image_url_https" />
              <div class="tweetUsernames">
                <span class="tweetName">{{ tweet.user.name }}</span>
                <span class="tweetUsername">@{{ tweet.user.screen_name }}</span>
              </div>
            </div>
            <div class="tweetBody">
              <span class="tweetText">{{ tweet.full_text }}</span>
              <div
                class="tweetMedia"
                v-if="tweet.extended_entities && tweet.extended_entities.media"
              >
                <li
                  v-for="media in tweet.extended_entities.media"
                  :key="media.id"
                >
                  <img
                    v-if="media.type === 'photo'"
                    class="tweetContent"
                    :src="media.media_url_https"
                  />
                  <video
                    v-if="
                      media.type === 'video' || media.type === 'animated_gif'
                    "
                    controls
                    class="tweetContent"
                  >
                    <source
                      :src="
                        media.video_info && media.video_info.variants[0].url
                      "
                      type="video/mp4"
                    />
                  </video>
                </li>
              </div>
              <span class="tweetCreatedAt">{{
                new Date(tweet.created_at).toLocaleString()
              }}</span>
            </div>
            <div class="tweetFooter">
              <div class="favorites">❤️{{ tweet.favorite_count }}</div>
              <div class="retweets">🔁{{ tweet.retweet_count }}</div>
            </div>
          </a>
        </div>
      </li>
    </ul>
  </div>
</template>

<script>
/* eslint-disable class-methods-use-this */
import { Component, Vue } from "vue-property-decorator";
import store from "@/store/index";
import constants from "@/store/constants";
import helpers from "@/util/helpers";
import remote from "electron";

const UserItemsPanelProps = Vue.extend({
  props: {
    itemtype: String
  }
});

@Component
export default class UserItemsPanel extends UserItemsPanelProps {
  currentPage = 1;

  perPage = 5;

  checkIfSelected(tweet) {
    return this.itemtype === "tweets"
      ? store.state.twitter[constants.WHITELISTED_TWEETS][`tweets-${tweet.id}`]
      : store.state.twitter[constants.WHITELISTED_FAVORITES][
          `favorites-${tweet.id}`
        ];
  }

  handleChanged(item) {
    if (this.itemtype === "tweets") {
      store.dispatch(constants.UPDATE_WHITELISTED_TWEETS, `tweets-${item.id}`);
    } else if (this.itemtype === "favorites") {
      store.dispatch(
        constants.UPDATE_WHITELISTED_FAVORITES,
        `favorites-${item.id}`
      );
    }
  }

  handleDeleteItemClicked(item) {
    if (
      // eslint-disable-next-line no-alert
      window.confirm(
        `Are you sure you want to delete this ${
          this.itemtype === "tweets" ? "tweet" : "❤️"
        }? THIS ACTION IS PERMANENT!`
      )
    ) {
      store.state.twitter[constants.TWITTER_USER_CLIENT]
        .post(
          this.itemtype === "tweets" ? "statuses/destroy" : "favorites/destroy",
          {
            id: item.id_str
          }
        )
        .then(() => {
          helpers.twitterGatherAndSetItems({
            apiRoute:
              this.itemtype === "tweets"
                ? constants.TWEETS_ROUTE
                : constants.FAVORITES_ROUTE,
            itemArray: []
          });
        });
    }
  }

  get loggedIn() {
    return store.state.twitter[constants.TWITTER_LOGGED_IN];
  }

  get userItems() {
    if (this.itemtype === "tweets") {
      return store.state.twitter[constants.USER_TWEETS].slice(
        (this.currentPage - 1) * this.perPage,
        this.currentPage * this.perPage
      );
    }
    if (this.itemtype === "favorites") {
      return store.state.twitter[constants.USER_FAVORITES].slice(
        (this.currentPage - 1) * this.perPage,
        this.currentPage * this.perPage
      );
    }
    return [];
  }

  get rows() {
    if (this.itemtype === "tweets") {
      return store.state.twitter[constants.USER_TWEETS].length;
    }
    if (this.itemtype === "favorites") {
      return store.state.twitter[constants.USER_FAVORITES].length;
    }
    return 0;
  }

  openExternalBrowser(link) {
    remote.shell.openExternal(link);
  }
}
</script>

<style lang="scss">
.tweetsContainer {
  width: 48%;
  height: 99%;
  border: 4mm ridge #1da1f2;
  margin-bottom: 10px;
}

.tweetList {
  padding-left: 0;
  list-style: none;
}

.tweetOptions {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.tweetWhitelist {
  display: flex;
  padding-left: 5px;
  padding-bottom: 20px;
}

.tweetDeleteIconContainer {
  :hover {
    cursor: pointer;
  }
}

.tweetDeleteIcon {
  width: 30px;
  height: 30px;
}

.tweetAndOptionsContainer {
  display: flex;
  align-items: center;
}

.tweet {
  padding: 15px;
  margin: 10px;
  border: 1px solid #e1e8ed;
  border-radius: 5px;
  flex-grow: 1;
  text-decoration: inherit;
  color: inherit;

  &:hover {
    background-color: #dddddd;
    text-decoration: inherit;
    color: inherit;
  }
}

.tweetHeader {
  display: flex;
  justify-content: flex-start;
}

.tweetUsernames {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  padding-left: 5px;
}

.tweetName {
  line-height: 1.3125;
  font-weight: bold;
}
.tweetUsername {
  color: #697882;
}

.tweetBody {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.tweetText {
  font-size: 24px;
}

.tweetCreatedAt {
  color: #697882;
}

.tweetMedia {
  display: grid;
  grid-template-columns: auto auto;
}

.tweetContent {
  border: 1px solid #e1e8ed;
  border-radius: 5px;
  width: 100%;
}

.tweetFooter {
  display: flex;
  justify-content: flex-start;
}

.retweets {
  padding-left: 20px;
}
</style>
