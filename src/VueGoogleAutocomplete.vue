<template>
  <div class="vue-google-autocomplete" style="position: relative;">
    <input
      ref="autocomplete"
      type="text"
      :class="classname"
      :id="id"
      :placeholder="placeholder"
      :disabled="disabled"
      v-model="autocompleteText"
      @focus="onFocus"
      @blur="onBlur"
      @input="onInput"
      @keydown.down.prevent="highlight(1)"
      @keydown.up.prevent="highlight(-1)"
      @keydown.enter.prevent="selectHighlighted"
    />
    <ul
      v-if="predictions.length"
      class="dropdown-menu show"
      style="position:absolute;top:100%;left:0;right:0;max-height:200px;overflow-y:auto;z-index:1000;"
    >
      <li
        v-for="(pred, idx) in predictions"
        :key="pred.placePrediction.placeId"
        :class="['dropdown-item',{ active: idx===highlightedIndex }]"
        @mousedown.prevent="select(pred)"
      >
        {{ pred.placePrediction.text.mainText }}
        <small v-if="pred.placePrediction.text.secondaryText"> — {{ pred.placePrediction.text.secondaryText }}</small>
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
    id:       { type: String, required: true },
    classname: String,
    placeholder: { type: String, default: 'Start typing' },
    disabled:   { type: Boolean, default: false },
    types:      { type: String, default: 'address' },
    fields:     { type: Array, default: () => ['address_components','formatted_address','geometry'] },
    country:    { type: [String,Array], default: null },
    enableGeolocation: { type: Boolean, default: false },
    geolocationOptions:{ type: Object, default: null }
  },
  data() {
    return {
      autocompleteText: '',
      predictions: [],
      highlightedIndex: -1,
      // Классы нового API
      AutocompleteSuggestion: null,
      AutocompleteSessionToken: null,
      Place: null
    };
  },
  async mounted() {
    // Ждём загрузки модуля places
    const places = await window.google.maps.importLibrary('places');
    this.AutocompleteSuggestion    = places.AutocompleteSuggestion;
    this.AutocompleteSessionToken  = places.AutocompleteSessionToken;
    this.Place                     = places.Place;
  },
  methods: {
    focus()   { this.$refs.autocomplete?.focus(); },
    blur()    { this.$refs.autocomplete?.blur(); },

    async onInput() {
      const q = this.autocompleteText;
      if (!q || !this.AutocompleteSuggestion) {
        this.predictions = [];
        return;
      }

      // Сессия — чтобы выбор и поиск шли вместе
      const sessionToken = new this.AutocompleteSessionToken();
      const req = {
        input: q,
        sessionToken,
        types: [this.types],
        componentRestrictions: this.country ? { country: this.country } : undefined
      };

      // Новый API: Promise → { suggestions }
      try {
        const { suggestions } =
          await this.AutocompleteSuggestion.fetchAutocompleteSuggestions(req);
        // suggestions: { placePrediction, }[]
        this.predictions = suggestions;
        this.highlightedIndex = -1;
        this.$emit('inputChange',{ newVal:q },this.id);
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
      // sugg.placePrediction → Place + fetchFields
      const pred = sugg.placePrediction;
      this.autocompleteText = pred.text.mainText + (pred.text.secondaryText ? ', '+pred.text.secondaryText : '');
      this.predictions = [];
      this.$emit('change',this.autocompleteText);

      // Конвертируем prediction в Place
      const place = pred.toPlace();
      await place.fetchFields({ fields: this.fields });

      // Форматируем и эмитим
      const data = this.formatResult(place);
      this.$emit('placechanged',data,place,this.id);
    },

    onFocus() {
      this.$emit('focus');
      if (this.enableGeolocation) this.geolocate();
    },
    onBlur() {
      setTimeout(() => this.predictions = [], 200);
      this.$emit('blur');
    },

    geolocate() {
      if (!navigator.geolocation) return;
      navigator.geolocation.getCurrentPosition(pos => {
        const loc = { lat:pos.coords.latitude, lng:pos.coords.longitude };
        new window.google.maps.Circle({ center:loc, radius:pos.coords.accuracy });
      }, err => this.$emit('error',err), this.geolocationOptions||{});
    },

    formatResult(place) {
      const out = {};
      (place.address_components||[]).forEach(c => {
        const t = c.types[0];
        if (ADDRESS_COMPONENTS[t]) out[t] = c[ADDRESS_COMPONENTS[t]];
      });
      if (place.geometry?.location) {
        out.latitude  = place.geometry.location.lat();
        out.longitude = place.geometry.location.lng();
      }
      out.formatted_address = place.formatted_address||'';
      return out;
    }
  }
};
</script>
