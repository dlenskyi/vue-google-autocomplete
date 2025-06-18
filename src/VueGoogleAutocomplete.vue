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
      @keypress="onKeyPress"
      @keyup="onKeyUp"
      style="width:100%; box-sizing:border-box;"
    />
    <ul
      v-if="predictions.length"
      class="dropdown-menu"
    >
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
      textValue: this.value || '',
      predictions: [],
      highlightedIndex: -1,
      AutocompleteSuggestion: null,
      AutocompleteSessionToken: null,
      Place: null
    };
  },

  computed: {
    // support v-model
    value: {
      get() { return this.textValue; },
      set(v) { this.textValue = v; }
    }
  },

  watch: {
    textValue(newVal, oldVal) {
      this.$emit('inputChange', { newVal, oldVal }, this.id);
    }
  },

  async mounted() {
    // load new places module
    const lib = await window.google.maps.importLibrary('places');
    this.AutocompleteSuggestion    = lib.AutocompleteSuggestion;
    this.AutocompleteSessionToken  = lib.AutocompleteSessionToken;
    this.Place                     = lib.Place;
    document.addEventListener('click', this.onClickOutside);
  },
  beforeDestroy() {
    document.removeEventListener('click', this.onClickOutside);
  },

  methods: {
    onClickOutside(e) {
      if (!this.$refs.root.contains(e.target)) {
        this.closeSuggestions();
      }
    },

    onInput() {
      // v-model updated already
      this.$emit('change', this.textValue);
      this.fetchSuggestions(this.textValue);
    },

    async fetchSuggestions(query) {
      if (!query || !this.AutocompleteSuggestion) {
        this.predictions = [];
        return;
      }
      const opts = { input: query, sessionToken: new this.AutocompleteSessionToken() };
      if (this.types) opts.types = [this.types];
      if (this.country) {
        opts.componentRestrictions = { country: this.country };
      }
      try {
        const { suggestions } =
          await this.AutocompleteSuggestion.fetchAutocompleteSuggestions(opts);
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
      let i = (this.highlightedIndex + delta + n) % n;
      this.highlightedIndex = i;
    },

    async selectHighlighted() {
      if (this.highlightedIndex >= 0) {
        await this.select(this.predictions[this.highlightedIndex]);
      }
    },

    async select(sugg) {
      const p = sugg.placePrediction;
      const txt = p.text.mainText + (p.text.secondaryText ? ', ' + p.text.secondaryText : '');
      // set text immediately
      this.$refs.autocomplete.value = txt;
      this.textValue = txt;
      this.$emit('input', txt);
      this.$emit('placechanged', null, null, this.id); // optional emitter for start
      this.closeSuggestions();

      // fetch details
      const place = p.toPlace();
      const fieldsReq = this.fields.map(f => {
        if (f === 'geometry') return 'location';
        if (f === 'url') return 'websiteURI';
        if (f === 'utc_offset_minutes') return 'utcOffsetMinutes';
        return f.replace(/_([a-z])/g, (_, c) => c.toUpperCase());
      });
      await place.fetchFields({ fields: fieldsReq });
      const result = this.formatResult(place);
      this.$emit('placechanged', result, place, this.id);
    },

    suggestionText(sugg) {
      const p = sugg.placePrediction;
      return p.text ? p.text.toString() : (p.description || '');
    },

    closeSuggestions() {
      this.predictions = [];
      this.highlightedIndex = -1;
    },

    onKeyPress(e) {
      this.$emit('keypress', e);
    },
    onKeyUp(e) {
      this.$emit('keyup', e);
    },

    formatResult(place) {
      const out = {};
      (place.addressComponents || []).forEach(c => {
        const type = c.types[0];
        if (ADDRESS_COMPONENTS[type]) out[type] = c[ADDRESS_COMPONENTS[type]];
      });
      if (place.geometry?.location) {
        out.latitude = place.geometry.location.lat();
        out.longitude = place.geometry.location.lng();
      }
      out.formatted_address = place.formattedAddress || '';
      out.url = place.websiteURI || '';
      return out;
    },

    geolocate() {
      if (!navigator.geolocation) return;
      navigator.geolocation.getCurrentPosition(pos => {
        const loc = { lat: pos.coords.latitude, lng: pos.coords.longitude };
        new window.google.maps.Circle({ center: loc, radius: pos.coords.accuracy });
      }, err => this.$emit('error', err), this.geolocationOptions || {});
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
.vue-google-autocomplete .dropdown-menu .dropdown-item {
  padding: 0 16px;
  height: 48px;
  line-height: 48px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  cursor: pointer;
  color: #3c4043;
}
.vue-google-autocomplete .dropdown-menu .dropdown-item.active,
.vue-google-autocomplete .dropdown-menu .dropdown-item:hover {
  background-color: #f1f3f4;
}
</style>
