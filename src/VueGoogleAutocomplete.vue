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
      style="
        position: absolute;
        top: 100%;
        left: 0;
        right: 0;
        max-height: 200px;
        overflow-y: auto;
        z-index: 1000;
      "
    >
      <li
        v-for="(p, idx) in predictions"
        :key="p.place_id"
        :class="['dropdown-item', { active: idx === highlightedIndex }]"
        @mousedown.prevent="select(p)"
      >
        {{ p.description }}
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
    types: { type: String, default: 'address' },
    fields: {
      type: Array,
      default: () => ['address_components', 'formatted_address', 'geometry']
    },
    country: { type: [String, Array], default: null },
    enableGeolocation: { type: Boolean, default: false },
    geolocationOptions: { type: Object, default: null }
  },

  data() {
    return {
      autocompleteText: '',
      predictions: [],
      highlightedIndex: -1,
      autocompleteService: null,
      placesService: null
    };
  },

  mounted() {
    // Ждём, пока google.maps.places будет доступен
    const initServices = () => {
      if (
        window.google &&
        window.google.maps &&
        window.google.maps.places &&
        !this.autocompleteService
      ) {
        this.autocompleteService =
          new window.google.maps.places.AutocompleteService();
        this.placesService =
          new window.google.maps.places.PlacesService(
            document.createElement('div')
          );
      } else if (!window.google || !window.google.maps || !window.google.maps.places) {
        // Если ещё не загружено, проверяем снова через 200ms
        setTimeout(initServices, 200);
      }
    };
    initServices();
  },

  methods: {
    focus() {
      this.$refs.autocomplete && this.$refs.autocomplete.focus();
    },
    blur() {
      this.$refs.autocomplete && this.$refs.autocomplete.blur();
    },

    onInput() {
      if (!this.autocompleteService) return;
      const input = this.autocompleteText;
      if (!input) {
        this.predictions = [];
        return;
      }
      const opts = { input, types: [this.types] };
      if (this.country) {
        opts.componentRestrictions = { country: this.country };
      }
      this.autocompleteService.getPlacePredictions(
        opts,
        (preds, status) => {
          if (
            status ===
            window.google.maps.places.PlacesServiceStatus.OK
          ) {
            this.predictions = preds;
            this.highlightedIndex = -1;
          } else {
            this.predictions = [];
          }
        }
      );
      this.$emit('inputChange', { newVal: input }, this.id);
    },

    highlight(delta) {
      const len = this.predictions.length;
      let idx = this.highlightedIndex + delta;
      if (idx < 0) idx = len - 1;
      if (idx >= len) idx = 0;
      this.highlightedIndex = idx;
    },

    selectHighlighted() {
      if (this.highlightedIndex >= 0) {
        this.select(this.predictions[this.highlightedIndex]);
      }
    },

    select(prediction) {
      this.autocompleteText = prediction.description;
      this.predictions = [];
      this.$emit('change', this.autocompleteText);
      if (!this.placesService) return;
      this.placesService.getDetails(
        {
          placeId: prediction.place_id,
          fields: this.fields
        },
        (place, status) => {
          if (
            status ===
            window.google.maps.places.PlacesServiceStatus.OK
          ) {
            const data = this.formatResult(place);
            this.$emit('placechanged', data, place, this.id);
          } else {
            this.$emit('error', status);
          }
        }
      );
    },

    onFocus() {
      this.$emit('focus');
      if (this.enableGeolocation) this.geolocate();
    },

    onBlur() {
      setTimeout(() => {
        this.predictions = [];
      }, 200);
      this.$emit('blur');
    },

    geolocate() {
      if (navigator.geolocation) {
        const options = this.geolocationOptions || {};
        navigator.geolocation.getCurrentPosition(
          pos => {
            const geoloc = { lat: pos.coords.latitude, lng: pos.coords.longitude };
            new window.google.maps.Circle({
              center: geoloc,
              radius: pos.coords.accuracy
            });
          },
          err => this.$emit('error', err),
          options
        );
      }
    },

    formatResult(place) {
      const rtn = {};
      if (place.address_components) {
        place.address_components.forEach(comp => {
          const type = comp.types[0];
          if (ADDRESS_COMPONENTS[type]) {
            rtn[type] = comp[ADDRESS_COMPONENTS[type]];
          }
        });
      }
      if (place.geometry && place.geometry.location) {
        rtn.latitude = place.geometry.location.lat();
        rtn.longitude = place.geometry.location.lng();
      }
      rtn.formatted_address = place.formatted_address || '';
      return rtn;
    }
  }
};
</script>

<style scoped>
.vue-google-autocomplete input.form-control {
  /* ваши стили для input */
}
.vue-google-autocomplete .dropdown-menu {
  /* ваши стили для выпадающего списка */
}
.vue-google-autocomplete .dropdown-item.active {
  /* стили для выделенного элемента */
}
</style>
