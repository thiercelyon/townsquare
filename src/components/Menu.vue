<template>
  <div id="controls">
    <span
      class="nomlog-summary"
      v-show="session.voteHistory.length && session.sessionId"
      @click="toggleModal('voteHistory')"
      :title="
        `${session.voteHistory.length} recent ${
          session.voteHistory.length == 1 ? 'nomination' : 'nominations'
        }`
      "
    >
      <font-awesome-icon icon="book-dead" />
      {{ session.voteHistory.length }}
    </span>
    <span
      class="session"
      :class="{
        spectator: session.isSpectator,
        reconnecting: session.isReconnecting
      }"
      v-if="session.sessionId"
      @click="leaveSession"
      :title="
        `${session.playerCount} ${$t('other-players-in-session')}${
          session.ping ? ' (' + session.ping + 'ms latency)' : ''
        }`
      "
    >
      <font-awesome-icon icon="broadcast-tower" />
      {{ session.playerCount }}
    </span>
    <div class="menu" :class="{ open: grimoire.isMenuOpen }">
      <font-awesome-icon icon="cog" @click="toggleMenu" />
      <ul>
        <li class="tabs" :class="tab">
          <font-awesome-icon icon="book-open" @click="tab = 'grimoire'" />
          <font-awesome-icon icon="broadcast-tower" @click="tab = 'session'" />
          <font-awesome-icon
            icon="users"
            v-if="!session.isSpectator"
            @click="tab = 'players'"
          />
          <font-awesome-icon icon="theater-masks" @click="tab = 'characters'" />
          <font-awesome-icon icon="question" @click="tab = 'help'" />
        </li>

        <template v-if="tab === 'grimoire'">
          <!-- Grimoire -->
          <li class="headline">{{ $t("grimoire") }}</li>
          <li @click="toggleGrimoire" v-if="players.length">
            <template v-if="!grimoire.isPublic">{{ $t("hide") }}</template>
            <template v-if="grimoire.isPublic">{{ $t("show") }}</template>
            <em>[G]</em>
          </li>
          <li @click="toggleNight" v-if="!session.isSpectator">
            <template v-if="!grimoire.isNight">
              {{ $t("switch-night") }}
            </template>
            <template v-if="grimoire.isNight">
              {{ $t("switch-day") }}
            </template>
            <em>[S]</em>
          </li>
          <li @click="toggleNightOrder" v-if="players.length">
            {{ $t("night-order") }}
            <em>
              <font-awesome-icon
                :icon="[
                  'fas',
                  grimoire.isNightOrder ? 'check-square' : 'square'
                ]"
              />
            </em>
          </li>
          <li v-if="players.length">
            {{ $t("zoom") }}
            <em>
              <font-awesome-icon
                @click="setZoom(grimoire.zoom - 1)"
                icon="search-minus"
              />
              {{ Math.round(100 + grimoire.zoom * 10) }}%
              <font-awesome-icon
                @click="setZoom(grimoire.zoom + 1)"
                icon="search-plus"
              />
            </em>
          </li>
          <li @click="setBackground">
            {{ $t("background-image") }}
            <em><font-awesome-icon icon="image"/></em>
          </li>
          <li v-if="!edition.isOfficial" @click="imageOptIn">
            <small>{{ $t("show-custom-image") }}</small>
            <em
              ><font-awesome-icon
                :icon="[
                  'fas',
                  grimoire.isImageOptIn ? 'check-square' : 'square'
                ]"
            /></em>
          </li>
          <li @click="toggleStatic">
            {{ $t("disable-animations") }}
            <em
              ><font-awesome-icon
                :icon="['fas', grimoire.isStatic ? 'check-square' : 'square']"
            /></em>
          </li>
          <li @click="toggleMuted">
            {{ $t("mute-sounds") }}
            <em
              ><font-awesome-icon
                :icon="['fas', grimoire.isMuted ? 'volume-mute' : 'volume-up']"
            /></em>
          </li>
        </template>

        <template v-if="tab === 'session'">
          <!-- Session -->
          <li class="headline" v-if="session.sessionId">
            {{ session.isSpectator ? $t("playing") : $t("hosting") }}
          </li>
          <li class="headline" v-else>
            {{ $t("live-session") }}
          </li>
          <template v-if="!session.sessionId">
            <li @click="hostSession">{{ $t("host") }}<em>[H]</em></li>
            <li @click="joinSession">{{ $t("join") }}<em>[J]</em></li>
          </template>
          <template v-else>
            <li v-if="session.ping">
              {{ $t("delay-to") }} {{ session.isSpectator ? "host" : "players" }}
              <em>{{ session.ping }}ms</em>
            </li>
            <li @click="copySessionUrl">
              {{ $t("copy-player-link") }}
              <em><font-awesome-icon icon="copy"/></em>
            </li>
            <li v-if="!session.isSpectator" @click="distributeRoles">
              {{ $t("send-characters") }}
              <em><font-awesome-icon icon="theater-masks"/></em>
            </li>
            <li
              v-if="session.voteHistory.length || !session.isSpectator"
              @click="toggleModal('voteHistory')"
            >
              {{ $t("vote-history") }}<em>[V]</em>
            </li>
            <li @click="leaveSession">
              {{ $t("leave-session") }}
              <em>{{ session.sessionId }}</em>
            </li>
          </template>
        </template>

        <template v-if="tab === 'players' && !session.isSpectator">
          <!-- Users -->
          <li class="headline">{{ $t("players") }}</li>
          <li @click="addPlayer" v-if="players.length < 20">
            {{ $t("add") }}<em>[A]</em>
          </li>
          <li @click="randomizeSeatings" v-if="players.length > 2">
            {{ $t("randomize") }}
            <em><font-awesome-icon icon="dice"/></em>
          </li>
          <li @click="clearPlayers" v-if="players.length">
            {{ $t("remove-all") }}
            <em><font-awesome-icon icon="trash-alt"/></em>
          </li>
        </template>

        <template v-if="tab === 'characters'">
          <!-- Characters -->
          <li class="headline">{{ $t("characters") }}</li>
          <li v-if="!session.isSpectator" @click="toggleModal('edition')">
            {{ $t("select-edition") }}
            <em>[E]</em>
          </li>
          <li
            @click="toggleModal('roles')"
            v-if="!session.isSpectator && players.length > 4"
          >
            {{ $t("choose-assign") }}
            <em>[C]</em>
          </li>
          <li v-if="!session.isSpectator" @click="toggleModal('fabled')">
            {{ $t("add-fabled") }}
            <em><font-awesome-icon icon="dragon"/></em>
          </li>
          <li @click="clearRoles" v-if="players.length">
            {{ $t("remove-all") }}
            <em><font-awesome-icon icon="trash-alt"/></em>
          </li>
        </template>

        <template v-if="tab === 'help'">
          <!-- Help -->
          <li class="headline">{{ $t("help") }}</li>
          <li @click="toggleModal('reference')">
            {{ $t("reference-sheet") }}
            <em>[R]</em>
          </li>
          <li @click="toggleModal('nightOrder')">
            {{ $t("night-order-sheet") }}
            <em>[N]</em>
          </li>
          <li @click="toggleModal('gameState')">
            {{ $t("game-state-json") }}
            <em><font-awesome-icon icon="file-code"/></em>
          </li>
          <li>
            <a href="https://discord.gg/Gd7ybwWbFk" target="_blank">
              {{ $t("join-discord") }}
            </a>
            <em>
              <a href="https://discord.gg/Gd7ybwWbFk" target="_blank">
                <font-awesome-icon :icon="['fab', 'discord']" />
              </a>
            </em>
          </li>
          <li>
            <a href="https://github.com/bra1n/townsquare" target="_blank">
              {{ $t("source-code") }}
            </a>
            <em>
              <a href="https://github.com/bra1n/townsquare" target="_blank">
                <font-awesome-icon :icon="['fab', 'github']" />
              </a>
            </em>
          </li>
        </template>
      </ul>
    </div>
  </div>
</template>

<script>
import { mapMutations, mapState } from "vuex";

export default {
  computed: {
    ...mapState(["grimoire", "session", "edition"]),
    ...mapState("players", ["players"])
  },
  data() {
    return {
      tab: "grimoire"
    };
  },
  methods: {
    setBackground() {
      const background = prompt(this.$t("enter-background"));
      if (background || background === "") {
        this.$store.commit("setBackground", background);
      }
    },
    hostSession() {
      if (this.session.sessionId) return;
      const sessionId = prompt(
        this.$t("create-session"),
        Math.round(Math.random() * 10000)
      );
      if (sessionId) {
        this.$store.commit("session/clearVoteHistory");
        this.$store.commit("session/setSpectator", false);
        this.$store.commit("session/setSessionId", sessionId);
        this.copySessionUrl();
      }
    },
    copySessionUrl() {
      const url = window.location.href.split("#")[0];
      const link = url + "#" + this.session.sessionId;
      navigator.clipboard.writeText(link);
    },
    distributeRoles() {
      if (this.session.isSpectator) return;
      const popup = this.$t("distribute-roles");
      if (confirm(popup)) {
        this.$store.commit("session/distributeRoles", true);
        setTimeout(
          (() => {
            this.$store.commit("session/distributeRoles", false);
          }).bind(this),
          2000
        );
      }
    },
    imageOptIn() {
      const popup = this.$t("custom-images-warn");
      if (this.grimoire.isImageOptIn || confirm(popup)) {
        this.toggleImageOptIn();
      }
    },
    joinSession() {
      if (this.session.sessionId) return this.leaveSession();
      let sessionId = prompt(this.$t("join-session"));
      if (sessionId.match(/^https?:\/\//i)) {
        sessionId = sessionId.split("#").pop();
      }
      if (sessionId) {
        this.$store.commit("session/clearVoteHistory");
        this.$store.commit("session/setSpectator", true);
        this.$store.commit("toggleGrimoire", false);
        this.$store.commit("session/setSessionId", sessionId);
      }
    },
    leaveSession() {
      if (confirm(this.$t("leave-warn"))) {
        this.$store.commit("session/setSpectator", false);
        this.$store.commit("session/setSessionId", "");
      }
    },
    addPlayer() {
      if (this.session.isSpectator) return;
      if (this.players.length >= 20) return;
      const name = prompt(this.$t("player-name"));
      if (name) {
        this.$store.commit("players/add", name);
      }
    },
    randomizeSeatings() {
      if (this.session.isSpectator) return;
      if (confirm(this.$t("randomize-warn"))) {
        this.$store.dispatch("players/randomize");
      }
    },
    clearPlayers() {
      if (this.session.isSpectator) return;
      if (confirm(this.$t("remove-players-warn"))) {
        // abort vote if in progress
        if (this.session.nomination) {
          this.$store.commit("session/nomination");
        }
        this.$store.commit("players/clear");
      }
    },
    clearRoles() {
      if (confirm(this.$t("remove-roles-warn"))) {
        this.$store.dispatch("players/clearRoles");
      }
    },
    toggleNight() {
      this.$store.commit("toggleNight");
      if (this.grimoire.isNight) {
        this.$store.commit("session/setMarkedPlayer", -1);
      }
    },
    ...mapMutations([
      "toggleGrimoire",
      "toggleMenu",
      "toggleImageOptIn",
      "toggleMuted",
      "toggleNightOrder",
      "toggleStatic",
      "setZoom",
      "toggleModal"
    ])
  }
};
</script>

<i18n>
  {
    "en": {
      "grimoire": "Grimoire",
      "zoom": "Zoom",
      "other-players-in-session": "other players in this session",
      "show-background-image": "Show background image",
      "background-image": "Background image",
      "show-custom-image": "Show custom image",
      "night-order": "Night order",
      "disable-animations": "Disable animations",
      "mute-sounds": "Mute sounds",
      "playing": "Playing",
      "hosting": "Hosting",
      "switch-night": "Switch to night",
      "switch-day": "Switch to day",
      "show": "Show",
      "hide": "Hide",
      "live-session": "Live Session",
      "delay-to": "Delay to",
      "vote-history": "Vote history",
      "leave-session": "Leave Session",
      "players": "Players",
      "add": "Add",
      "randomize": "Randomize",
      "remove-all": "Remove all",
      "characters": "Characters",
      "select-edition": "Select Edition",
      "choose-assign": "Choose & Assign",
      "add-fabled": "Add Fabled",
      "help": "Help",
      "reference-sheet": "Reference Sheet",
      "night-order-sheet": "Night Order Sheet",
      "game-state-json": "Game State JSON",
      "join-discord": "Join Discord",
      "source-code": "Source code",
      "host": "Host (Storyteller)",
      "join": "Join (Player)",
      "copy-player-link": "Copy player link",
      "send-characters": "Send characters",
      "player-name": "Player name",
      "enter-background": "Enter custom background URL",
      "create-session": "Enter a channel number / name for your session",
      "distribute-roles": "Do you want to distribute assigned characters to all SEATED players?",
      "custom-images-warn": "Are you sure you want to allow custom images? A malicious script file author might track your IP address this way.",
      "join-session": "Enter the channel number / name of the session you want to join",
      "leave-warn": "Are you sure you want to leave the active live game?",
      "randomize-warn": "Are you sure you want to randomize seatings?",
      "remove-players-warn": "Are you sure you want to remove all players?",
      "remove-roles-warn": "Are you sure you want to remove all player roles?"
    },
    "fr": {
      "grimoire": "Grimoire",
      "zoom": "Zoom",
      "other-players-in-session": "autres joueurs dans la partie",
      "show-background-image": "Montrer l'image de fond",
      "background-image": "Image de fond",
      "show-custom-image": "Afficher image de fond",
      "night-order": "Ordre de nuit",
      "disable-animations": "Désactiver les animations",
      "mute-sounds": "Couper le son",
      "playing": "Joueur",
      "hosting": "Hôte",
      "switch-night": "Passer à la nuit",
      "switch-day": "Passer à la journée",
      "show": "Afficher",
      "hide": "Masquer",
      "live-session": "Session",
      "players": "Joueur",
      "add": "Ajouter",
      "randomize": "Randomiser",
      "remove-all": "Tout supprimer",
      "characters": "Personnages",
      "select-edition": "Choisir Module",
      "choose-assign": "Choisir & Assigner",
      "add-fabled": "Ajouter Légendaire",
      "help": "Aide",
      "reference-sheet": "Fiche de Rôles",
      "night-order-sheet": "Ordre de Nuit",
      "game-state-json": "JSON Etat du Jeu",
      "join-discord": "Rejoindre Discord",
      "source-code": "Code source",
      "delay-to": "Retarder de",
      "vote-history": "Historique des votes",
      "leave-session": "Quitter",
      "host": "Héberger (Conteur)",
      "join": "Rejoindre (Joueur)",
      "copy-player-link": "Copier lien de partie",
      "send-characters": "Envoyer rôles",
      "player-name": "Nom du joueur",
      "enter-background": "Entrez l'URL de l'image de fond à charger",
      "create-session": "Saisir l'identifiant de session / nom de votre session",
      "distribute-roles": "Souhaitez-vous assigner ces rôles à TOUS les joueurs assis ?",
      "custom-images-warn": "Êtes-vous certain de vouloir autoriser les images personnalisées ? Un auteur script malintentionné pourrait obtenir votre adresse IP de cette façon.",
      "join-session": "Saisir l'identifiant de session / nom de la session à rejoindre",
      "leave-warn": "Êtes-vous certain de vouloir quitter la partie en cours ?",
      "randomize-warn": "Êtes-vous certain de vouloir randomiser le positionnement des joueurs ?",
      "remove-players-warn": "Êtes-vous certain de vouloir supprimer tous les joueurs ?",
      "remove-roles-warn": "Êtes-vous certain de vouloir supprimer tous les rôles ?"
    }
  }
</i18n>

<style scoped lang="scss">
@import "../vars.scss";

// success animation
@keyframes greenToWhite {
  from {
    color: green;
  }
  to {
    color: white;
  }
}

// Controls
#controls {
  position: absolute;
  right: 3px;
  top: 3px;
  text-align: right;
  padding-right: 50px;
  z-index: 75;

  svg {
    filter: drop-shadow(0 0 5px rgba(0, 0, 0, 1));
    &.success {
      animation: greenToWhite 1s normal forwards;
      animation-iteration-count: 1;
    }
  }

  > span {
    display: inline-block;
    cursor: pointer;
    z-index: 5;
    margin-top: 7px;
    margin-left: 10px;
  }

  span.nomlog-summary {
    color: $townsfolk;
  }

  span.session {
    color: $demon;
    &.spectator {
      color: $townsfolk;
    }
    &.reconnecting {
      animation: blink 1s infinite;
    }
  }
}

@keyframes blink {
  50% {
    opacity: 0.5;
    color: gray;
  }
}

.menu {
  width: 220px;
  transform-origin: 200px 22px;
  transition: transform 500ms cubic-bezier(0.68, -0.55, 0.27, 1.55);
  transform: rotate(-90deg);
  position: absolute;
  right: 0;
  top: 0;

  &.open {
    transform: rotate(0deg);
  }

  > svg {
    cursor: pointer;
    background: rgba(0, 0, 0, 0.5);
    border: 3px solid black;
    width: 40px;
    height: 50px;
    margin-bottom: -8px;
    border-bottom: 0;
    border-radius: 10px 10px 0 0;
    padding: 5px 5px 15px;
  }

  a {
    color: white;
    text-decoration: none;
    &:hover {
      color: red;
    }
  }

  ul {
    display: flex;
    list-style-type: none;
    padding: 0;
    margin: 0;
    flex-direction: column;
    overflow: hidden;
    box-shadow: 0 0 10px black;
    border: 3px solid black;
    border-radius: 10px 0 10px 10px;

    li {
      padding: 2px 5px;
      color: white;
      text-align: left;
      background: rgba(0, 0, 0, 0.7);
      display: flex;
      align-items: center;
      justify-content: space-between;
      min-height: 30px;

      &.tabs {
        display: flex;
        padding: 0;
        svg {
          flex-grow: 1;
          flex-shrink: 0;
          height: 35px;
          border-bottom: 3px solid black;
          border-right: 3px solid black;
          padding: 5px 0;
          cursor: pointer;
          transition: color 250ms;
          &:hover {
            color: red;
          }
          &:last-child {
            border-right: 0;
          }
        }
        &.grimoire .fa-book-open,
        &.players .fa-users,
        &.characters .fa-theater-masks,
        &.session .fa-broadcast-tower,
        &.help .fa-question {
          background: linear-gradient(
            to bottom,
            $townsfolk 0%,
            rgba(0, 0, 0, 0.5) 100%
          );
        }
      }

      &:not(.headline):not(.tabs):hover {
        cursor: pointer;
        color: red;
      }

      em {
        flex-grow: 0;
        font-style: normal;
        margin-left: 10px;
        font-size: 80%;
      }
    }

    .headline {
      font-family: PiratesBay, sans-serif;
      letter-spacing: 1px;
      padding: 0 10px;
      text-align: center;
      justify-content: center;
      background: linear-gradient(
        to right,
        $townsfolk 0%,
        rgba(0, 0, 0, 0.5) 20%,
        rgba(0, 0, 0, 0.5) 80%,
        $demon 100%
      );
    }
  }
}
</style>
