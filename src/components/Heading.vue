<template>
  <div class="stuff">
    <div class="desktop-lang-switcher">
      <div class="desktop-lang-switch-button" @click="toggleLangSwitcher">{{ lang.languageShort }}</div>
      <div class="lang-switcher" :class="langSwitcherOpen ? 'open' : 'closed'">
        <div
          class="lang"
          v-for="language in languages"
          :key="language"
          @click="switchLang(language)"
        >{{ language }}</div>
      </div>
    </div>
    <div class="header">
      <h1>
        Animal Crossing:
        <br />New&nbsp;Horizon
        <br />Helper
      </h1>
    </div>

    <div class="main-content-wrapper">
      <!-- MOBILE -->
      <MobileHeading
        v-if="isMobile"
        :choice="choice"
        :importExportActive="importExportActive"
        :importExport="importExport"
        :exportError="exportError"
        :importError="importError"
        :showSuccess="showSuccess"
        @choose="choose"
        @handleImport="handleImport"
        @handleExport="handleExport"
      />
      <!-- DESKTOP -->
      <DesktopHeading
        v-else
        :choice="choice"
        :importExportActive="importExportActive"
        :importExport="importExport"
        :exportError="exportError"
        :importError="importError"
        :showSuccess="showSuccess"
        @choose="choose"
        @handleImport="handleImport"
        @handleExport="handleExport"
        @toggleFeedback="toggleFeedback"
      />
    </div>
  </div>
</template>
<script>
import MobileHeading from "./MobileHeading.vue";
import DesktopHeading from "./DesktopHeading.vue";
import store from "../store/index.js";
import { mapState } from "vuex";
import { queryForIndexes, queryByIndex, mobileCheck } from "../utils/helper.js";

export default {
  data() {
    return {
      exportError: false,
      importError: false,
      importExport: "",
      feedbackOpenClose: "",
      isMobile: false,
      importExportActive: false,
      showSuccess: false,
      langSwitcherOpen: false
    };
  },
  components: {
    MobileHeading,
    DesktopHeading
  },
  mounted() {
    this.isMobile = mobileCheck();
  },
  watch: {
    importError(val) {
      if (!!val) {
        setTimeout(() => {
          this.importError = false;
        }, 2000);
      }
    },
    exportError(val) {
      if (!!val) {
        setTimeout(() => {
          this.exportError = false;
        }, 2000);
      }
    }
  },
  methods: {
    switchLang(lang) {
      store.dispatch("switchLang", lang);
      this.toggleLangSwitcher();
    },
    toggleLangSwitcher() {
      this.langSwitcherOpen = !this.langSwitcherOpen;
    },
    choose(data) {
      this.importError = false;
      this.exportError = false;
      this.importExportActive = false;
      store.commit("assignDisplayData", data);
      store.dispatch("storeSelection");
    },
    toggleFeedback() {
      this.importError = false;
      this.exportError = false;
      this.feedbackOpenClose = this.feedbackOpenClose.length > 0 ? "" : "open";
    },
    handleExport() {
      this.importError = false;
      this.exportError = false;
      this.importExportActive = true;
      const isOldData = localStorage.getItem("myfishies");
      const pattern = /([a-z])([A-Z])/g;
      let newData;

      // determine whether the savedata is legacy data and save accordingly
      if (!!isOldData) {
        newData = [
          {
            caught: {
              fish: localStorage
                .getItem("myfishies")
                .replace(pattern, "$1,$2")
                .split(","),
              insects: localStorage
                .getItem("myinsecties")
                .replace(pattern, "$1,$2")
                .split(","),
              songs: localStorage
                .getItem("songs")
                .replace(pattern, "$1,$2")
                .split(",")
            }
          }
        ];
      } else {
        newData = [
          {
            caught: {
              fish: JSON.parse(localStorage.getItem("caught"))
                ? JSON.parse(localStorage.getItem("caught")).fish
                : [],
              insects: JSON.parse(localStorage.getItem("caught"))
                ? JSON.parse(localStorage.getItem("caught")).insects
                : [],
              songs: JSON.parse(localStorage.getItem("caught"))
                ? JSON.parse(localStorage.getItem("caught")).songs
                : []
            }
          }
        ];
      }
      // get golden-tool counts
      let counts = {
        shovelCount: localStorage.getItem("shovelCount") || 0,
        axeCount: localStorage.getItem("axeCount") || 0,
        balloonCount: localStorage.getItem("balloonCount") || 0,
        wateringcanCount: localStorage.getItem("wateringcanCount") || 0
      };
      // save all to newData
      newData = [...newData, counts];
      let exportFish, exportInsects, exportSongs;
      // create the export string for fish and insects
      if (!newData[0].caught) {
        this.exportError = true;
        return;
      }
      exportFish = queryForIndexes(
        newData[0].caught.fish,
        this.fish
      ).toString();
      exportInsects = queryForIndexes(
        newData[0].caught.insects,
        this.insects
      ).toString();
      exportSongs = queryForIndexes(
        newData[0].caught.songs,
        this.songs
      ).toString();
      // only pass values if there are any fish or insects to begin with
      exportFish = exportFish.length > 0 ? "f:" + exportFish + "|" : "";
      exportInsects =
        exportInsects.length > 0 ? "i:" + exportInsects + "|" : "";
      exportSongs = exportSongs.length > 0 ? "s:" + exportSongs + "|" : "";
      let exportCounts =
        (counts.shovelCount == 0 ? "" : "sc:" + counts.shovelCount + "|") +
        (counts.axeCount == 0 ? "" : "ac:" + counts.axeCount + "|") +
        (counts.balloonCount == 0 ? "" : "bc:" + counts.balloonCount + "|") +
        (counts.wateringcanCount == 0
          ? ""
          : "wc:" + counts.wateringcanCount + "|");
      const exportString =
        exportFish + exportInsects + exportSongs + exportCounts;
      if (exportString == "|" || exportString == "") {
        this.exportError = true;
        return;
      } else {
        this.exportError = false;
      }
      this.importExport = exportString;
    },
    handleImport(data) {
      this.importError = false;
      this.exportError = false;
      if ((this.isMobile && !this.importExportActive) || data.length === 0) {
        this.importExportActive = true;
        return;
      }
      const importValue = data;
      this.importExport = "";
      const fishPattern = /f:([\d,]*)\|?.*/;
      const insectsPattern = /.*i:([\d,]*)\|?.*/;
      const songsPattern = /.*s:([\d]*)\|?.*/;
      const axeCountPattern = /ac:(\d)/;
      const shovelCountPattern = /sc:(\d)/;
      const balloonCountPattern = /bc:(\d)/;
      const wateringcanCountPattern = /wc:(\d)/;
      let fish = [];
      let insects = [];
      let songs = [];

      if (fishPattern.test(importValue))
        fish = importValue.replace(fishPattern, "$1").split(",");
      if (insectsPattern.test(importValue))
        insects = importValue.replace(insectsPattern, "$1").split(",");
      if (songsPattern.test(importValue))
        songs = importValue.replace(songsPattern, "$1").split(",");

      let axeCount = axeCountPattern.exec(importValue) || 0;
      axeCount = !!axeCount ? axeCount[1] : 0;
      let shovelCount = shovelCountPattern.exec(importValue) || 0;
      shovelCount = !!shovelCount ? shovelCount[1] : 0;
      let balloonCount = balloonCountPattern.exec(importValue) || 0;
      balloonCount = !!balloonCount ? balloonCount[1] : 0;
      let wateringcanCount = wateringcanCountPattern.exec(importValue) || 0;
      wateringcanCount = !!wateringcanCount ? wateringcanCount[1] : 0;

      const caught = {
        fish: queryByIndex(fish, this.fish),
        insects: queryByIndex(insects, this.insects),
        songs: queryByIndex(songs, this.songs)
      };

      localStorage.setItem("caught", JSON.stringify(caught));
      localStorage.setItem("axeCount", axeCount);
      localStorage.setItem("shovelCount", shovelCount);
      localStorage.setItem("balloonCount", balloonCount);
      localStorage.setItem("wateringcanCount", wateringcanCount);
      store.dispatch("getData");
      this.importExportActive = false;
      this.showSuccess = true;
      setTimeout(() => {
        this.showSuccess = false;
      }, 1500);
    }
  },
  computed: {
    ...mapState({
      choice: "displayData",
      fish: "fish",
      insects: "insects",
      songs: "songs",
      lang: "lang",
      languages: "languages"
    })
  }
};
</script>