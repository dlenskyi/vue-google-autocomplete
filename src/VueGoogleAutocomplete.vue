<template>
  <div class="vue-google-autocomplete" ref="root" style="position: relative; width: 100%;">
    <input
      ref="autocomplete"
      :id="id"
      type="text"
      :class="classname"
      :placeholder="placeholder"
      :disabled="disabled"
      v-model="textValue"
      @input="onInput"
      @keydown.down.prevent="highlight(1)"
      @keydown.up.prevent="highlight(-1)"
      @keydown.enter.prevent="selectHighlighted"
      @keydown.esc.prevent="closeSuggestions"
      style="width:100%; box-sizing:border-box;"
    />
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
const ADDRESS_COMPONENTS = {
  subpremise: 'short_name',
  street_number: 'short_name',
  route: 'long_name',
  locality: 'long_name',
  administrative_area_level_1: 'short_name',
  administrative_area_level_2: 'long_name',
  country: 'long_name',
  postal_code: 'short_name'
};

export default {
  name: 'VueGoogleAutocomplete',

  props: {
    value: { type: String, default: '' },
    id: { type: String, required: true },
    classname: String,
    placeholder: { type: String, default: 'Start typing' },
    disabled: { type: Boolean, default: false },
    types: { type: String, default: null },
    fields: { type: Array, default: () => ['address_components','formatted_address','geometry','url','utc_offset_minutes'] },
    country: { type: [String, Array], default: null },
    enableGeolocation: { type: Boolean, default: false },
    geolocationOptions: { type: Object, default: null }
  },

  data() {
    return {
      textValue: this.value,
      predictions: [],
      highlightedIndex: -1,
      AutocompleteSuggestion: null,
      AutocompleteSessionToken: null,
      Place: null
    };
  },

  watch: {
    value(newVal) {
      this.textValue = newVal;
    }
  },

  async mounted() {
    const lib = await window.google.maps.importLibrary('places');
    this.AutocompleteSuggestion    = lib.AutocompleteSuggestion;
    this.AutocompleteSessionToken  = lib.AutocompleteSessionToken;
    this.Place                     = lib.Place;
    document.addEventListener('click', this.onClickOutside);
  },
  beforeDestroy() {
    document.removeEventListener('click', this.onClickOutside);
  },

  computed: {
    // two-way binding for v-model
    textValue: {
      get() {
        return this.value;
      },
      set(v) {
        this.$emit('input', v);
      }
    }
  },

  methods: {
    onClickOutside(e) {
      if (!this.$refs.root.contains(e.target)) {
        this.closeSuggestions();
      }
    },

    onInput() {
      this.fetchSuggestions(this.textValue);
    },

    async fetchSuggestions(query) {
      if (!query || !this.AutocompleteSuggestion) {
        this.predictions = [];
        return;
      }
      const opts = { input: query, sessionToken: new this.AutocompleteSessionToken() };
      if (this.types) opts.types = [this.types];
      // remove componentRestrictions: not supported by AutocompleteSuggestion
      try {
        const { suggestions } = await this.AutocompleteSuggestion.fetchAutocompleteSuggestions(opts);
        this.predictions = suggestions;
        this.highlightedIndex = -1;
      } catch (err) {
        console.error('AutocompleteSuggestion error', err);
        this.$emit('error', err);
        this.predictions = [];
      }
    },

    highlight(delta) {
      const n = this.predictions.length;
      if (!n) return;
      let i = this.highlightedIndex + delta;
      if (i < 0) i = n - 1;
      if (i >= n) i = 0;
      this.highlightedIndex = i;
    },

    async selectHighlighted() {
      if (this.highlightedIndex >= 0) {
        await this.select(this.predictions[this.highlightedIndex]);
      }
    },

    async select(sugg) {
      const p = sugg.placePrediction;
      const fullText = p.text.mainText + (p.text.secondaryText ? ', ' + p.text.secondaryText : '');
      // set immediately
      this.$refs.autocomplete.value = fullText;
      this.$emit('input', fullText);
      this.predictions = [];
      this.highlightedIndex = -1;

      const place = p.toPlace();
      const fieldsReq = this.fields.map(f => {
        if (f === 'geometry') return 'location';
        if (f === 'url') return 'websiteURI';
        if (f === 'utc_offset_minutes') return 'utcOffsetMinutes';
        return f.replace(/_([a-z])/g, (_, c) => c.toUpperCase());
      });
      await place.fetchFields({ fields: fieldsReq });
      const data = this.formatResult(place);
      this.$emit('placechanged', data, place, this.id);
    },

    closeSuggestions() {
      this.predictions = [];
      this.highlightedIndex = -1;
    },

    getSuggestionText(sugg) {
      const p = sugg.placePrediction;
      return p.text ? p.text.toString() : (p.description || '');
    },

    formatResult(place) {
      const out = {};
      (place.addressComponents || []).forEach(c => {
        const type = c.types[0];
        if (ADDRESS_COMPONENTS[type]) out[type] = c[ADDRESS_COMPONENTS[type]];
      });
      if (place.location) {
        out.latitude = place.location.lat();
        out.longitude = place.location.lng();
      }
      out.formatted_address = place.formattedAddress || '';
      out.url = place.websiteURI || '';
      return out;
    },

    closeSuggestions() {
      this.predictions = [];
      this.highlightedIndex = -1;
    }
  }
};
</script>

<style scoped>
.vue-google-autocomplete .dropdown-menu {
  background: #fff;
  border: 1px solid #dadce0;
  border-radius: 4px;
  box-shadow: 0 2px 6px rgba(32,33,36,.28);
  font-family: Roboto, Arial, sans-serif;
  font-size: 16px;
  margin-top: 4px;
  padding: 0;
  position: absolute;
  width: 100%;
  z-index: 1000;
  list-style: none;
}
.vue-google-autocomplete .dropdown-item {
  padding: 0 16px;
  height: 48px;
  line-height: 48px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  cursor: pointer;
}
.vue-google-autocomplete .dropdown-item.active,
.vue-google-autocomplete .dropdown-item:hover {
  background-color: #f1f3f4;
}
</style>
