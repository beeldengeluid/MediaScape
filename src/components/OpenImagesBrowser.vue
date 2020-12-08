<template>
  <div
    :oncontextmenu="touchMode ? 'return false;' : 'return true;'"
    :class="{ 'user-select-none': touchMode }"
  >
    <TheHeader class="mb4" :class="{ mt5: touchMode }" />
    <TheIntro class="mb5" v-if="!touchMode && !state.surveyMode" />
    <div class="flex flex-wrap justify-between items-end">
      <TheCTA class="f3" :touchMode="touchMode" />
      <StateStory
        :state="state"
        :computed="{
          selectedDecade: selectedDecade,
          activeLength: itemsFilteredSorted.length,
          totalLength: items.length,
        }"
        @toggle-active-filter="onToggleActiveFilter"
        class="f3"
      />
    </div>
    <div class="tr">
      <v-btn @click="randomizeSelection" outlined class="ml-auto self-end mt3">
        <v-icon left>mdi-shuffle</v-icon>Randomize Selection
      </v-btn>
    </div>
    <PeriodChart
      :barSeries="decadeCounts"
      :lineSeries="decadeCountsForSelection"
      @decade-click="onDecadeClick"
      :decadeIndex="state.decadeIndex"
      :colors="{
        bar: colors.inactive,
        line: colors.secondary,
        background: colors.background,
      }"
    />
    <v-container fluid class="pa0">
      <v-row>
        <v-col cols="12" lg="2">
          <h3 class="mb3">
            <span class="bb b--secondary">
              <v-icon>mdi-tag</v-icon> Subjects in selection
              <span class="fw1">{{
                filtersForSelection["subjects"].length
              }}</span></span
            >
          </h3>
          <FilterList
            :filters="filtersForSelection['subjects']"
            :activeFilters="state.activeFilters['subjects']"
            @toggle-active-filter="
              onToggleActiveFilter({
                type: 'subjects',
                value: $event.value,
              })
            "
            @toggle-tail="onToggleTail"
            activeClass="teal"
          />
        </v-col>
        <v-col cols="12" lg="2">
          <h3 class="mb3">
            <span class="bb b--secondary">
              <v-icon>mdi-map-marker</v-icon> Locations in selection
              <span class="fw1">{{
                filtersForSelection["locations"].length
              }}</span></span
            >
          </h3>
          <FilterList
            :filters="filtersForSelection['locations']"
            :activeFilters="state.activeFilters['locations']"
            @toggle-active-filter="
              onToggleActiveFilter({
                type: 'locations',
                value: $event.value,
              })
            "
            @toggle-tail="onToggleTail"
            activeClass="teal"
          />
        </v-col>
        <v-col cols="12" lg="8">
          <div class="mb3 flex flex-wrap">
            <h3 class="dib mr3 mb3">
              <span class="bb b--secondary mr2"
                >Videos in selection
                <span class="fw1">{{ itemsFilteredSorted.length }}</span></span
              >
              <span class="fw1 grey--text bb b--primary-accent"
                >(of {{ decades[state.decadeIndex].count }} in decade)</span
              >
            </h3>
            <v-btn
              @click="resetState"
              v-show="hasActiveFilters"
              outlined
              class="ml-auto"
            >
              <v-icon left>mdi-close-circle</v-icon>Clear Selection
            </v-btn>
          </div>
          <div class="flex flex-wrap mb3 items-center">
            <div class="flex items-center mr3 mr4-l">
              <span class="mr2 fw7">Sort by</span>
              <v-chip-group
                v-model="state.sortBy"
                active-class="orange darken-2"
                mandatory
                class="fw5 font-mono"
              >
                <v-chip
                  v-for="sortField in sortFields"
                  :key="sortField"
                  :value="sortField"
                >
                  {{ sortField }}
                </v-chip>
              </v-chip-group>
              <v-btn fab x-small color="orange darken-2">
                <v-icon @click="toggleSortAscending">{{
                  state.sortAscending ? "mdi-chevron-up" : "mdi-chevron-down"
                }}</v-icon>
              </v-btn>
            </div>
            <div class="flex items-center mr3 mr4-l">
              <span class="pr2 fw7">Display</span>
              <v-chip-group
                v-model="state.displayFieldsSelected"
                active-class="orange darken-2"
                multiple
                class="fw5 font-mono"
              >
                <v-chip
                  v-for="displayField in displayFields"
                  :key="displayField"
                  :value="displayField"
                >
                  {{ displayField }}
                </v-chip>
              </v-chip-group>
            </div>
            <v-btn
              @click="openPlaylist"
              color="orange darken-2"
              class="ml-auto"
            >
              <v-icon left>mdi-playlist-play</v-icon>Start Playlist
            </v-btn>
          </div>
          <ZoomSlider
            v-model="zoom.value"
            :min="zoom.min"
            :max="zoom.max"
            :step="zoom.step"
            :tickLabels="zoomLabels"
            color="orange darken-2"
            class="mb3 mb4-ns"
          ></ZoomSlider>
          <CollectionItemGrid
            :items="itemsFilteredSorted"
            :noThumbsPerRow="noThumbsPerRow"
            :displayFieldsSelected="state.displayFieldsSelected"
            :activeFilters="state.activeFilters"
            :filterCountsForSelection="filterCountsForSelection"
            :touchMode="touchMode"
            @toggle-active-filter="onToggleActiveFilter"
            @open-playlist-at="openPlaylist"
            @open-transcript-at="openTranscript"
          />
        </v-col>
      </v-row>
    </v-container>

    <back-to-top>
      <v-btn light title="Back to top" absolute bottom right fab>
        <v-icon>mdi-chevron-up</v-icon>
      </v-btn>
    </back-to-top>

    <v-snackbar
      v-model="snackbar.state"
      :timeout="snackbar.timeout"
      color="grey lighten-4 grey grey--text text--darken-4"
    >
      <span v-html="snackbar.markup"></span>
    </v-snackbar>

    <ModalContainer v-show="state.showPlaylist" @close-modal="closePlaylist">
      <VideoPlaylist
        :items="itemsFilteredSorted"
        :stretchVideo="false"
        :filterCountsForSelection="filterCountsForSelection"
        :activeFilters="state.activeFilters"
        :isShown="state.showPlaylist"
        @preview-click="loadPlaylist"
        color="orange darken-2"
        class="h-100 justify-center"
        ref="videoPlaylist"
      />
    </ModalContainer>
    <ModalContainer v-if="state.showTranscript" @close-modal="closeTranscript">
      <ASRComposition
        :videoSrc="itemsFilteredSorted[state.transcriptIndex].videoSrc"
        :thumbSrc="itemsFilteredSorted[state.transcriptIndex].thumbSrc"
        :nerSequences="itemsFilteredSorted[state.transcriptIndex].layer__ner"
        :asrSequences="itemsFilteredSorted[state.transcriptIndex].layer__asr"
        :isShown="state.showTranscript"
        ref="asrComposition"
      />
    </ModalContainer>
  </div>
</template>

<script>
import _ from "lodash";
import TheHeader from "@/components/TheHeader";
import TheIntro from "@/components/TheIntro";
import TheCTA from "@/components/TheCTA";
import StateStory from "@/components/StateStory";
import PeriodChart from "@/components/PeriodChart";
import FilterList from "@/components/FilterList";
import ZoomSlider from "@/components/ZoomSlider";
import CollectionItemGrid from "@/components/CollectionItemGrid";
import ModalContainer from "@/components/ModalContainer";
import VideoPlaylist from "@/components/VideoPlaylist";
import ASRComposition from "@/components/ASRComposition";
import BackToTop from "vue-backtotop";

export default {
  components: {
    TheHeader,
    TheIntro,
    TheCTA,
    StateStory,
    PeriodChart,
    FilterList,
    ZoomSlider,
    CollectionItemGrid,
    ModalContainer,
    VideoPlaylist,
    ASRComposition,
    BackToTop,
  },
  data() {
    return {
      state: {
        decadeIndex: 6,
        sortBy: "date",
        sortAscending: true,
        displayFieldsSelected: ["thumb", "title", "year"],
        activeFilters: {
          locations: [],
          subjects: [],
        },
        showPlaylist: false,
        showTranscript: false,
        playlistIndex: 0,
        surveyMode: false,
        transcriptIndex: 0,
      },
      zoom: {
        value: 3,
        min: 0,
        max: 6,
        step: 1,
      },
      chartSeries: [],
      snackbar: {
        state: false,
        text: "",
        timeout: 4000,
      },
    };
  },
  props: {
    items: { type: Array, default: () => [] },
    touchMode: { type: Boolean, default: false },
    sortFields: { type: Array, default: () => ["id", "date", "title"] },
    displayFields: { type: Array, default: () => ["title", "year", "thumb"] },
    filterFields: { type: Array, default: () => ["locations", "subjects"] },
    colors: { 
      type: Object, 
      default: () => ({
        primary: "#5E35B1",
        secondary: "#009688",
        background: "#121212",
        inactive: "#555",
      }),
    },
  },
  static: {
    defaultState: {
      activeFilters: {
        locations: [],
        subjects: [],
      },
    },
  },
  computed: {
    itemsPerDecade() {
      return _.groupBy(this.items, (i) => this.dateToDecade(i["date"]));
    },
    noThumbsPerRow() {
      return Math.pow(2, this.zoom.value);
    },
    zoomLabels() {
      return _.range(this.zoom.max + 1).map((value) => Math.pow(2, value));
    },
    itemsFiltered() {
      return this.itemsPerDecade[
        this.decades[this.state.decadeIndex].name
      ].filter((i) =>
        this.filterFields.every((filterField) =>
          this.filterByFilterField(filterField, i)
        )
      );
    },
    itemsFilteredAllDecades() {
      return this.items.filter((i) =>
        this.filterFields.every((filterField) =>
          this.filterByFilterField(filterField, i)
        )
      );
    },
    itemsFilteredSorted() {
      let order = this.state.sortAscending ? "asc" : "desc";
      return _.orderBy(this.itemsFiltered, [this.state.sortBy], [order]);
    },
    selectedYearRange() {
      let decadeYearMin = parseInt(
        this.decades[this.state.decadeIndex].name.slice(0, 4)
      );
      let decadeYearMax = decadeYearMin + 9;
      return [decadeYearMin, decadeYearMax];
    },
    selectedDecade() {
      return this.decades[this.state.decadeIndex].name;
    },
    yearMin() {
      return Math.min(...this.items.map((i) => i["date"].slice(0, 4)));
    },
    yearMax() {
      return Math.max(...this.items.map((i) => i["date"].slice(0, 4)));
    },
    decadeMin() {
      return Math.floor(this.yearMin / 10) * 10;
    },
    decadeMax() {
      return Math.floor(this.yearMax / 10) * 10;
    },
    filterCountsForSelection() {
      const getCounts = (items, fieldName) => {
        let names = _.flatMap(items, (i) => i[fieldName]);
        return _.countBy(names);
      };

      let filterCounts = {};
      this.filterFields.forEach((filterField) => {
        filterCounts[filterField] = getCounts(
          this.itemsFilteredSorted,
          filterField
        );
      });
      return filterCounts;
    },
    filtersForSelection() {
      const getFiltersOrdered = (filterCounts) => {
        let filtersForSelection = this.objectToCollection(
          filterCounts,
          "name",
          "count"
        );
        return _.orderBy(
          filtersForSelection,
          ["count", "name"],
          ["desc", "asc"]
        );
      };

      return this.filterFields.reduce(
        (filtersAccumulator, currentFilterField) => {
          filtersAccumulator[currentFilterField] = getFiltersOrdered(
            this.filterCountsForSelection[currentFilterField]
          );
          return filtersAccumulator;
        },
        {}
      );
    },
    hasActiveFilters() {
      return _.flatten(_.values(this.state.activeFilters)).length > 0;
    },
    decadeCounts() {
      return this.getDecadeCounts(this.items, this.decadeMin, this.decadeMax);
    },
    decades() {
      return this.objectToCollection(this.decadeCounts, "name", "count");
    },
    decadeCountsForSelection() {
      return this.getDecadeCounts(
        this.itemsFilteredAllDecades,
        this.decadeMin,
        this.decadeMax
      );
    },
  },
  methods: {
    objectToCollection(countsObject, keyForKey, keyForValue) {
      return Object.keys(countsObject).map((k) => {
        return {
          [keyForKey]: k,
          [keyForValue]: countsObject[k],
        };
      });
    },
    toggleSortAscending() {
      this.state.sortAscending = !this.state.sortAscending;
    },
    dateToYear(date) {
      return date.slice(0, 4);
    },
    dateToDecade(date) {
      return date.slice(0, 3) + "0s";
    },
    onToggleActiveFilter(event) {
      if (this.state.activeFilters[event.type].includes(event.value)) {
        this.removeActiveFilter(event.type, event.value);
      } else {
        this.addActiveFilter(event.type, event.value);
      }
    },
    addActiveFilter(filterType, filterValue) {
      /* for unclear reason .push() doesn't register correctly in the watch()
         function, so using .concat() instead */
      this.state.activeFilters[filterType] = this.state.activeFilters[
        filterType
      ].concat([filterValue]);
    },
    removeActiveFilter(filterType, filterValue) {
      this.state.activeFilters[filterType] = this.state.activeFilters[
        filterType
      ].filter((lf) => lf !== filterValue);
    },
    filterByYear(item) {
      return (
        this.dateToYear(item["date"]) >= this.selectedYearRange[0] &&
        this.dateToYear(item["date"]) <= this.selectedYearRange[1]
      );
    },
    filterByFilterField(FilterField, item) {
      return (
        this.state.activeFilters[FilterField].every((lf) =>
          item[FilterField].includes(lf)
        ) || !this.state.activeFilters[FilterField].length
      );
    },
    onDecadeClick(dataPointIndex) {
      // set decade
      this.state.decadeIndex = dataPointIndex;
    },
    getDecadeCounts(items, decadeMin, decadeMax) {
      // get decades present in data
      let decadesPresent = _.countBy(items, (i) =>
        this.dateToDecade(i["date"])
      );

      // add intermediary decades
      _.range(decadeMin, decadeMax + 10, 10).map((d) => {
        let decade = d + "s";
        if (!Object.keys(decadesPresent).includes(decade)) {
          decadesPresent[decade] = 0;
        }
      });

      // sort keys
      const decadesSorted = {};
      Object.keys(decadesPresent)
        .sort()
        .forEach(function(key) {
          decadesSorted[key] = decadesPresent[key];
        });

      return decadesSorted;
    },
    showSnackbar(markup) {
      this.snackbar.state = false;
      this.$nextTick(() => {
        this.snackbar.markup = markup;
        this.snackbar.state = true;
      });
    },
    onToggleTail(newValue) {
      if (newValue) {
        this.showSnackbar(
          `👀 Showing the <strong>full list of filters</strong>`
        );
      } else {
        this.showSnackbar(
          `🙈 Hiding <strong>filters with 1 occurance</strong>`
        );
      }
    },
    qsCustomizer(objValue, srcValue) {
      return typeof objValue === "number"
        ? parseInt(srcValue, 10)
        : typeof objValue === "boolean"
        ? srcValue === "1"
        : srcValue;
    },
    resetState() {
      this.state.activeFilters = Object.assign(
        {},
        this.$options.static.defaultState.activeFilters
      );
    },
    randomItemFromArray(array) {
      return array[Math.floor(Math.random() * array.length)];
    },
    randomizeDecade() {
      let decadeIndicesAvailable = _.range(_.size(this.decades)).filter(
        (index) =>
          this.decades[index].count > 0 && index !== this.state.decadeIndex
      );
      this.state.decadeIndex = this.randomItemFromArray(decadeIndicesAvailable);
    },
    randomizeFilter(filterType) {
      let filtersAvailable = this.filtersForSelection[filterType].filter(
        (filter) => filter.count > 1
      );
      if (filtersAvailable.length) {
        let randomFilter = this.randomItemFromArray(filtersAvailable).name;
        this.state.activeFilters[filterType] = [randomFilter];
        return true;
      } else {
        return false;
      }
    },
    randomizeSelection() {
      this.resetState();
      this.randomizeDecade();

      let isRandomized = this.randomizeFilter(
        this.filterFields[_.random(1)]
      );
      if (!isRandomized) {
        // no filters in this decade, recursively retry
        this.randomizeSelection();
      }
    },
    addHTMLClass(className) {
      const htmlEl = document.getElementsByTagName("html")[0];
      htmlEl.classList.add(className);
    },
    removeHTMLClass(className) {
      const htmlEl = document.getElementsByTagName("html")[0];
      htmlEl.classList.remove(className);
    },
    openPlaylist(event) {
      if (typeof event === "number") {
        this.$refs.videoPlaylist.setCurrentItemIndex(event);
      }
      this.state.showPlaylist = true;
      this.addHTMLClass("overflow-y-hidden");
    },
    openTranscript(event) {
      if (typeof event === "number") {
        this.state.transcriptIndex = event;
      }
      this.state.showTranscript = true;
      this.addHTMLClass("overflow-y-hidden");
    },
    closePlaylist() {
      this.state.showPlaylist = false;
      this.$refs.videoPlaylist.pauseVideo();
      this.removeHTMLClass("overflow-y-hidden");
    },
    closeTranscript() {
      this.state.showTranscript = false;
      this.$refs.asrComposition.pauseVideo();
      this.removeHTMLClass("overflow-y-hidden");
    },
    loadPlaylist({ type, value }) {
      this.state.activeFilters = Object.assign(
        {},
        this.$options.static.defaultState.activeFilters,
        { [type]: [value] }
      );
    },
    handleFilterUpdate(newValue, oldValue, filterType) {
      let added = _.difference(newValue, oldValue);
      if (added.length) {
        this.showSnackbar(
          `👓 Added ${filterType} filter <strong>${added[0]}</strong>`
        );
      } else {
        let removed = _.difference(oldValue, newValue);
        if (removed.length) {
          this.showSnackbar(
            `❌ Removed ${filterType} filter <strong>${removed[0]}</strong>`
          );
        }
      }

      this.$router
        .push({
          query: Object.assign({}, this.$route.query, {
            activeFilters: {
              ...this.state.activeFilters,
              [filterType]: newValue,
            },
          }),
        })
        .catch((err) => {
          // Ignore the vuex err regarding  navigating to the page they are already on.
          if (
            err.name !== "NavigationDuplicated" &&
            !err.message.includes(
              "Avoided redundant navigation to current location"
            )
          ) {
            // But print any other errors to the console
            // eslint-disable-next-line
            console.error(err);
          }
        });
    },
  },
  watch: {
    "state.sortBy": function(newValue) {
      this.showSnackbar(
        `${
          this.state.sortAscending ? "☝️" : "👇"
        } Sorting by <strong>${newValue}</strong>`
      );
      this.$router.push({
        query: Object.assign({}, this.$route.query, { sortBy: newValue }),
      });
    },
    "state.decadeIndex": function(newValue) {
      this.showSnackbar(
        `✅ Selected <strong>${this.decades[newValue].name}</strong> decade`
      );
      this.$router.push({
        query: Object.assign({}, this.$route.query, { decadeIndex: newValue }),
      });
    },
    "state.sortAscending": function(newValue) {
      this.showSnackbar(
        `${newValue ? "☝️" : "👇"} Sorting in <strong>${
          newValue ? "ascending" : "descending"
        }</strong> order`
      );
      this.$router.push({
        query: Object.assign({}, this.$route.query, {
          sortAscending: newValue,
        }),
      });
    },
    "state.activeFilters.locations": function(newValue, oldValue) {
      this.handleFilterUpdate(newValue, oldValue, "locations");
    },
    "state.activeFilters.subjects": function(newValue, oldValue) {
      this.handleFilterUpdate(newValue, oldValue, "subjects");
    },
    "state.displayFieldsSelected": function(newValue, oldValue) {
      let added = _.difference(newValue, oldValue);
      if (added.length) {
        this.showSnackbar(`👀 Displaying <strong>${added[0]}</strong>`);
      } else {
        let removed = _.difference(oldValue, newValue);
        if (removed.length) {
          this.showSnackbar(`🙈 Not displaying <strong>${removed[0]}</strong>`);
        }
      }
      this.$router.push({
        query: Object.assign({}, this.$route.query, {
          displayFieldsSelected: newValue,
        }),
      });
    },
  },
  created() {
    let queryParams = _.assignWith(
      this.state,
      this.$route.query,
      this.qsCustomizer
    );
    // ensure activeFilters has all default keys defined in case not in queryParams
    this.state.activeFilters = Object.assign(
      {},
      this.$options.static.defaultState.activeFilters,
      queryParams.activeFilters
    );
  },
  mounted() {
    if (this.touchMode) {
      document
        .querySelector("meta[name='viewport']")
        .setAttribute(
          "content",
          "width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no"
        );
    }
  },
};
</script>
<style>
.user-select-none {
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  -o-user-select: none;
  user-select: none;
}
.bb {
  border-bottom-width: 3px !important;
}
</style>