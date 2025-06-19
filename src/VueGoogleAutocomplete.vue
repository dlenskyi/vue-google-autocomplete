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
      @keydown.enter="onEnter"
    />
    <ul
      v-if="predictions.length"
      class="dropdown-menu show"
      style="position: absolute; top: 100%; left: 0; right: 0; max-height: 200px; overflow-y: auto; z-index: 1000;"
    >
      <li
        v-for="(sugg, idx) in predictions"
        :key="sugg.placePrediction.placeId"
        :class="['dropdown-item', { active: idx === highlightedIndex }]"
        @click.prevent="select(sugg)"
      >
        {{ sugg.placePrediction.text
            ? sugg.placePrediction.text.toString()
            : (sugg.placePrediction.description || '') }}
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
    country: { type: [String, Array], default: null },
    fields: { type: Array, default: () => ['address_components', 'formatted_address', 'geometry', 'url', 'utc_offset_minutes'] },
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
        this.$emit('input', val);
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
    async onEnter(e) {
      if (this.predictions.length > 0 && this.highlightedIndex >= 0) {
        e.preventDefault();
        await this.selectHighlighted();
      }
    },
    
    focus() {
      this.$refs.autocomplete && this.$refs.autocomplete.focus();
    },
    blur() {
      this.$refs.autocomplete && this.$refs.autocomplete.blur();
    },

    async fetchSuggestions(query) {
      if (!query || !this.AutocompleteSuggestion) {
        this.predictions = [];
        return;
      }
    
      const sessionToken = new this.AutocompleteSessionToken();
      const req = { input: query, sessionToken };
    
      if (this.country) {
        // Нормализуем массив кодов в верхний регистр
        const codes = (Array.isArray(this.country)
          ? this.country
          : [this.country]
        ).map(c => c.toUpperCase());
    
        // Жёстко ограничиваем выдачу этими странами
        req.includedRegionCodes = codes;
      }
    
      try {
        const { suggestions } =
          await this.AutocompleteSuggestion.fetchAutocompleteSuggestions(req);
        this.predictions     = suggestions;
        this.highlightedIndex = -1;
      } catch (e) {
        console.error("AutocompleteSuggestion error", e);
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
        // после удачного select очистим и predictions, и highlightedIndex
        this.predictions = [];
        this.highlightedIndex = -1;
      }
    },

    async select(sugg) {
      const pred = sugg.placePrediction;
      const fullText = sugg.placePrediction.text.text;
      this.$emit('input', fullText);
      this.predictions = [];

      const place = pred.toPlace();
      // convert snake_case fields to camelCase for fetch
      const fieldsToRequest = this.fields.map(field => {
        switch (field) {
          case 'geometry':
            return 'location';
          case 'url':
            return 'websiteURI';
          default:
            return field.replace(/_([a-z])/g, (_, c) => c.toUpperCase());
        }
      });
      await place.fetchFields({ fields: fieldsToRequest });

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
      (place.addressComponents || []).forEach(comp => {
        const type = comp.types[0];
        const keyMap = {
          subpremise: 'short_name',
          street_number: 'short_name',
          route: 'long_name',
          locality: 'long_name',
          administrative_area_level_1: 'short_name',
          administrative_area_level_2: 'long_name',
          country: 'long_name',
          postal_code: 'short_name'
        };
        if (keyMap[type]) out[type] = comp[keyMap[type]];
      });
      if (place.location) {
        out.latitude = place.location.lat();
        out.longitude = place.location.lng();
      }
      out.formatted_address = place.formattedAddress || '';
      out.url = place.websiteURI || '';
      return out;
    }
  }
};
</script>

<style scoped>
.vue-google-autocomplete .dropdown-menu {
  background: #fff;
  border: 1px solid #dadce0;
  border-radius: 4px;
  box-shadow: 0 2px 6px rgba(32, 33, 36, .28);
  font-family: Roboto, Arial, sans-serif;
  font-size: 16px;
  margin-top: 4px;
  padding: 0;
  position: absolute;
  width: 100%;
  z-index: 1000;
  overflow: hidden;
}

.vue-google-autocomplete .dropdown-item {
  display: flex;
  align-items: center;
  height: 48px;
  padding: 0 16px;
  cursor: pointer;
  color: #3c4043;
  line-height: 20px;
  white-space: nowrap;
  text-overflow: ellipsis;
  overflow: hidden;
}

/* маркер-иконка слева */
.vue-google-autocomplete .dropdown-item::before {
  content: '';
  flex: none;
  width: 20px;
  height: 20px;
  margin-right: 12px;
  background-image: url("data:image/svg+xml;charset=UTF-8,<svg fill='%23666' height='20' viewBox='0 0 24 24' width='20' xmlns='http://www.w3.org/2000/svg'><path d='M12 2C8.13 2 5 5.13 5 9c0 5.25 7 13 7 13s7-7.75 7-13c0-3.87-3.13-7-7-7z'/><circle cx='12' cy='9' fill='%23fff' r='2.5'/></svg>");
  background-size: 20px 20px;
}

/* hover и текущий пункт */
.vue-google-autocomplete .dropdown-item:hover,
.vue-google-autocomplete .dropdown-item.active {
  background-color: #f1f3f4;
}

/* «powered by Google» внизу */
.vue-google-autocomplete .dropdown-menu::after {
  content: 'powered by Google';
  display: block;
  padding: 8px 16px;
  font-size: 12px;
  color: #70757a;
  text-align: right;
}
</style>
