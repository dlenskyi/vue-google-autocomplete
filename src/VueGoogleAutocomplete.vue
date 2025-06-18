<template>
  <div class="vue-google-autocomplete" style="position: relative; width: 100%;">
    <input
      ref="input"
      :id="id"
      :class="classname"
      :placeholder="placeholder"
      :disabled="disabled"
      v-model="textValue"
      @input="onInput"
      @keydown.down.prevent="highlight(1)"
      @keydown.up.prevent="highlight(-1)"
      @keydown.enter.prevent="selectHighlighted"
      style="width:100%; box-sizing:border-box;"
    />
    <ul v-if="predictions.length" class="dropdown-menu">
      <li
        v-for="(pred, idx) in predictions"
        :key="pred.place_id"
        :class="['dropdown-item', { active: idx === highlightedIndex }]"
        @mousedown.prevent="select(pred)"
      >
        {{ pred.description }}
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
    fields: {
      type: Array,
      default: () => [
        'address_components',
        'formatted_address',
        'geometry',
        'url',
        'utc_offset_minutes'
      ]
    },
    country: { type: [String, Array], default: null }
  },
  data() {
    return {
      predictions: [],
      highlightedIndex: -1,
      autocompleteService: null,
      placesService: null
    };
  },
  computed: {
    textValue: {
      get() {
        return this.value;
      },
      set(v) {
        this.$emit('input', v);
      }
    }
  },
  watch: {
    value(val) {
      // keep internal v-model in sync
      // (textValue setter will emit back, so avoid loop)
      if (val !== this.textValue) {
        this.textValue = val;
      }
    }
  },
  mounted() {
    // Инициализируем старые сервисы AutocompleteService/PlacesService
    const init = () => {
      if (
        window.google &&
        window.google.maps &&
        window.google.maps.places &&
        !this.autocompleteService
      ) {
        this.autocompleteService = new google.maps.places.AutocompleteService();
        this.placesService = new google.maps.places.PlacesService(
          document.createElement('div')
        );
      } else {
        setTimeout(init, 200);
      }
    };
    init();
  },
  methods: {
    onInput() {
      // при вводе текста запрашиваем подсказки
      this.fetchSuggestions(this.textValue);
    },
    fetchSuggestions(input) {
      if (!input || !this.autocompleteService) {
        this.predictions = [];
        return;
      }
      const opts = { input };
      if (this.types) {
        opts.types = [this.types];
      }
      if (this.country) {
        opts.componentRestrictions = {
          country: this.country
        };
      }
      this.autocompleteService.getPlacePredictions(
        opts,
        (preds, status) => {
          if (
            status ===
            google.maps.places.PlacesServiceStatus.OK
          ) {
            this.predictions = preds;
          } else {
            this.predictions = [];
          }
          this.highlightedIndex = -1;
        }
      );
    },
    highlight(delta) {
      const n = this.predictions.length;
      if (!n) return;
      let i = this.highlightedIndex + delta;
      if (i < 0) i = n - 1;
      if (i >= n) i = 0;
      this.highlightedIndex = i;
    },
    selectHighlighted() {
      if (this.highlightedIndex >= 0) {
        this.select(this.predictions[this.highlightedIndex]);
      }
    },
    select(prediction) {
      // сразу ставим description
      const text = prediction.description;
      this.$refs.input.value = text;
      this.$emit('input', text);
      // скрываем список
      this.predictions = [];
      this.highlightedIndex = -1;

      // получаем детали через PlacesService
      this.placesService.getDetails(
        {
          placeId: prediction.place_id,
          fields: this.fields.map(f => {
            if (f === 'geometry') return 'geometry';
            return f;
          })
        },
        (place, status) => {
          if (
            status ===
            google.maps.places.PlacesServiceStatus.OK
          ) {
            const data = this.formatResult(place);
            this.$emit('placechanged', data, place, this.id);
          } else {
            this.$emit('error', status);
          }
        }
      );
    },
    formatResult(place) {
      const out = {};
      (place.address_components || []).forEach(c => {
        const type = c.types[0];
        if (ADDRESS_COMPONENTS[type]) {
          out[type] = c[ADDRESS_COMPONENTS[type]];
        }
      });
      if (place.geometry && place.geometry.location) {
        out.latitude = place.geometry.location.lat();
        out.longitude = place.geometry.location.lng();
      }
      out.formatted_address = place.formatted_address || '';
      out.url = place.url || '';
      return out;
    },
    getSuggestionText(pred) {
      return pred.description;
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
  cursor: pointer;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  color: #3c4043;
}
.vue-google-autocomplete .dropdown-item:hover,
.vue-google-autocomplete .dropdown-item.active {
  background-color: #f1f3f4;
}
</style>
