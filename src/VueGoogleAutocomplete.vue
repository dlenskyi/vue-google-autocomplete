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
      @focus="onFocus"
      @blur="onBlur"
      @keydown.down.prevent="highlight(1)"
      @keydown.up.prevent="highlight(-1)"
      @keydown.enter.prevent="selectHighlighted"
    />
    <ul
      v-if="predictions.length"
      class="dropdown-menu show"
      style="position:absolute; top:100%; left:0; right:0; max-height:200px; overflow-y:auto; z-index:1000;"
    >
      <li
        v-for="(sugg, idx) in predictions"
        :key="sugg.placePrediction.placeId"
        :class="['dropdown-item', { active: idx === highlightedIndex }]"
        @mousedown.prevent="select(sugg)"
      >
        {{ sugg.placePrediction.text.mainText }}
        <small v-if="sugg.placePrediction.text.secondaryText">
          — {{ sugg.placePrediction.text.secondaryText }}
        </small>
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

  // Добавляем поддержку v-model (Vue 2): prop 'value' и $emit('input')
  props: {
    value: { type: String, default: '' },
    id: { type: String, required: true },
    classname: String,
    placeholder: { type: String, default: 'Start typing' },
    disabled: { type: Boolean, default: false },
    country: { type: [String, Array], default: null },
    fields: { type: Array, default: () => ['address_components', 'formatted_address', 'geometry'] },
    enableGeolocation: { type: Boolean, default: false },
    geolocationOptions: { type: Object, default: null }
  },

  data() {
    return {
      predictions: [],
      highlightedIndex: -1,
      AutocompleteSuggestion: null,
      AutocompleteSessionToken: null,
      Place: null
    };
  },

  computed: {
    textValue: {
      get() {
        return this.value;
      },
      set(val) {
        this.$emit('input', val);                   // for v-model binding
        this.fetchSuggestions(val);
      }
    }
  },

  async mounted() {
    const places = await window.google.maps.importLibrary('places');
    this.AutocompleteSuggestion   = places.AutocompleteSuggestion;
    this.AutocompleteSessionToken = places.AutocompleteSessionToken;
    this.Place                    = places.Place;
  },

  methods: {
    focus() { this.$refs.autocomplete?.focus(); },
    blur()  { this.$refs.autocomplete?.blur(); },

    async fetchSuggestions(query) {
      if (!query || !this.AutocompleteSuggestion) {
        this.predictions = [];
        return;
      }

      const sessionToken = new this.AutocompleteSessionToken();
      const req = { input: query, sessionToken };
      if (this.country) {
        if (Array.isArray(this.country)) req.includedRegionCodes = this.country;
        else req.region = this.country;
      }

      try {
        const { suggestions } = await this.AutocompleteSuggestion.fetchAutocompleteSuggestions(req);
        this.predictions = suggestions;
        this.highlightedIndex = -1;
      } catch (e) {
        console.error('AutocompleteSuggestion error', e);
        this.predictions = [];
      }
    },

    highlight(delta) {
      const len = this.predictions.length;
      let idx = this.highlightedIndex + delta;
      if (idx < 0) idx = len - 1;
      if (idx >= len) idx = 0;
      this.highlightedIndex = idx;
    },

    async selectHighlighted() {
      if (this.highlightedIndex >= 0) {
        await this.select(this.predictions[this.highlightedIndex]);
      }
    },

    async select(sugg) {
      const pred = sugg.placePrediction;
      const fullText = pred.text.mainText + (pred.text.secondaryText ? ', ' + pred.text.secondaryText : '');
      this.$emit('input', fullText);
      this.predictions = [];

      const place = pred.toPlace();
      await place.fetchFields({ fields: this.fields });
      const data = this.formatResult(place);
      this.$emit('placechanged', data, place, this.id);
    },

    onFocus() {
      this.$emit('focus');
      if (this.enableGeolocation) this.geolocate();
    },

    onBlur() {
      setTimeout(() => (this.predictions = []), 200);
      this.$emit('blur');
    },

    geolocate() {
      if (!navigator.geolocation) return;
      navigator.geolocation.getCurrentPosition(
        pos => {
          const loc = { lat: pos.coords.latitude, lng: pos.coords.longitude };
          new window.google.maps.Circle({ center: loc, radius: pos.coords.accuracy });
        },
        err => this.$emit('error', err),
        this.geolocationOptions || {}
      );
    },

    formatResult(place) {
      const out = {};
      (place.address_components || []).forEach(c => {
        const t = c.types[0];
        if (ADDRESS_COMPONENTS[t]) out[t] = c[ADDRESS_COMPONENTS[t]];
      });
      if (place.geometry?.location) {
        out.latitude = place.geometry.location.lat();
        out.longitude = place.geometry.location.lng();
      }
      out.formatted_address = place.formatted_address || '';
      return out;
    }
  }
};
</script>

<style scoped>
.vue-google-autocomplete input.form-control {
  /* ваши стили */
}
.vue-google-autocomplete .dropdown-menu {
  /* ваши стили */
}
.vue-google-autocomplete .dropdown-item.active {
  /* ваши стили */
}
</style>
