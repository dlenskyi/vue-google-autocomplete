<template>
  <div class="vue-google-autocomplete" style="position: relative;">
    <input
      ref="autocomplete"
      type="text"
      :class="classname"
      :id="id"
      :placeholder="placeholder"
      :disabled="disabled"
      v-model="textValue"
      @input="onInput"
      @keydown.down.prevent="highlight(1)"
      @keydown.up.prevent="highlight(-1)"
      @keydown.enter.prevent="selectHighlighted"
    />
    <!-- dropdown -->
    <ul v-if="predictions.length" class="dropdown-menu">
      <li
        v-for="(sugg, idx) in predictions"
        :key="sugg.placePrediction.placeId"
        :class="['dropdown-item', { active: idx === highlightedIndex }]"
        @mousedown.prevent="select(sugg)"
      >
        {{ getSuggestionText(sugg) }}
      </li>
    </ul>
  </div>
</template>

<script>
export default {
  name: "VueGoogleAutocomplete",
  props: {
    value: { type: String, default: "" },
    id: { type: String, required: true },
    classname: String,
    placeholder: { type: String, default: "Start typing" },
    disabled: { type: Boolean, default: false },
    fields: {
      type: Array,
      default: () => [
        "address_components",
        "formatted_address",
        "geometry",
        "url",
        "utc_offset_minutes",
      ],
    },
  },
  data() {
    return {
      textValue: this.value,
      predictions: [],
      highlightedIndex: -1,
      AutocompleteSuggestion: null,
      AutocompleteSessionToken: null,
      Place: null,
    };
  },
  watch: {
    value(val) {
      this.textValue = val;
    },
  },
  async mounted() {
    // грузим новый Data-API
    const places = await window.google.maps.importLibrary("places");
    this.AutocompleteSuggestion = places.AutocompleteSuggestion;
    this.AutocompleteSessionToken = places.AutocompleteSessionToken;
    this.Place = places.Place;
  },
  methods: {
    onInput() {
      // пушим в v-model и запрашиваем подсказки
      this.$emit("input", this.textValue);
      this.fetchSuggestions(this.textValue);
    },
    async fetchSuggestions(input) {
      if (!input || !this.AutocompleteSuggestion) {
        this.predictions = [];
        return;
      }
      const token = new this.AutocompleteSessionToken();
      try {
        const { suggestions } =
          await this.AutocompleteSuggestion.fetchAutocompleteSuggestions({
            input,
            sessionToken: token,
          });
        this.predictions = suggestions;
        this.highlightedIndex = -1;
      } catch (e) {
        console.error("AutocompleteSuggestion error", e);
        this.predictions = [];
      }
    },
    highlight(delta) {
      if (!this.predictions.length) return;
      let i = this.highlightedIndex + delta;
      if (i < 0) i = this.predictions.length - 1;
      if (i >= this.predictions.length) i = 0;
      this.highlightedIndex = i;
    },
    async selectHighlighted() {
      if (this.highlightedIndex >= 0) {
        await this.select(this.predictions[this.highlightedIndex]);
      }
    },
    async select(sugg) {
      // Собираем текст
      const p = sugg.placePrediction;
      const fullText =
        p.text.mainText +
        (p.text.secondaryText ? ", " + p.text.secondaryText : "");
      // Устанавливаем сразу в v-model
      this.textValue = fullText;
      this.$emit("input", fullText);
      // Скрываем список
      this.predictions = [];
      this.highlightedIndex = -1;

      // Получаем детали места
      const place = p.toPlace();
      const fieldsToRequest = this.fields.map((f) => {
        if (f === "geometry") return "location";
        if (f === "url") return "websiteURI";
        if (f === "utc_offset_minutes") return "utcOffsetMinutes";
        return f.replace(/_([a-z])/g, (_, c) => c.toUpperCase());
      });
      await place.fetchFields({ fields: fieldsToRequest });

      const data = this.formatResult(place);
      this.$emit("placechanged", data, place, this.id);
    },
    getSuggestionText(sugg) {
      const p = sugg.placePrediction;
      return p.text
        ? p.text.toString()
        : p.description || "";
    },
    formatResult(place) {
      const out = {};
      (place.addressComponents || []).forEach((c) => {
        const t = c.types[0];
        const map = {
          subpremise: "short_name",
          street_number: "short_name",
          route: "long_name",
          locality: "long_name",
          administrative_area_level_1: "short_name",
          administrative_area_level_2: "long_name",
          country: "long_name",
          postal_code: "short_name",
        };
        if (map[t]) out[t] = c[map[t]];
      });
      if (place.location) {
        out.latitude = place.location.lat();
        out.longitude = place.location.lng();
      }
      out.formatted_address = place.formattedAddress || "";
      out.url = place.websiteURI || "";
      return out;
    },
  },
};
</script>

<style scoped>
.vue-google-autocomplete .dropdown-menu {
  background: #fff;
  border: 1px solid #dadce0;
  border-radius: 4px;
  box-shadow: 0 2px 6px rgba(32, 33, 36, 0.28);
  font-family: Roboto, Arial, sans-serif;
  font-size: 16px;
  margin-top: 4px;
  padding: 0;
  position: absolute;
  width: 100%;
  z-index: 1000;
}
.vue-google-autocomplete .dropdown-item {
  padding: 0 16px;
  height: 48px;
  line-height: 48px;
  cursor: pointer;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  color: #3c4043;
}
.vue-google-autocomplete .dropdown-item.active,
.vue-google-autocomplete .dropdown-item:hover {
  background-color: #f1f3f4;
}
</style>
