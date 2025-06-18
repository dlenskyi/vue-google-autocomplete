<template>
  <div ref="autocompleteContainer"></div>
</template>

<script>
/**
 * VueGoogleAutocomplete
 * A Vue.js autosuggest component for the Google Places API.
 *
 * Props:
 *  - id (String, required): Element ID.
 *  - classname (String): CSS class for input.
 *  - placeholder (String): Placeholder text.
 *  - disabled (Boolean): Whether input is disabled.
 *  - types (String): Autocomplete types.
 *  - fields (Array): PlaceResult fields to fetch.
 *  - country (String|Array): Component restrictions.
 *  - enableGeolocation (Boolean): Bias to user location.
 *  - geolocationOptions (Object): Options for geolocation.
 *
 * Emits:
 *  - inputChange({ newVal, oldVal }, id)
 *  - placechanged(placeResult, placePrediction, id)
 *  - focus, blur, change, keypress, keyup
 *  - error(status)
 *
 * Methods:
 *  - geolocate(): Use browser geolocation.
 *  - updateCoordinates(value): Reverse geocode coordinates.
 *  - biasAutocompleteLocation(): Bias predictions by location.
 *
 * See: https://github.com/olefirenko/vue-google-autocomplete
 */

// Mapping Google address component keys to object properties
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

const CITIES_TYPE = ['locality', 'administrative_area_level_3'];
const REGIONS_TYPE = ['locality', 'sublocality', 'postal_code', 'country',
  'administrative_area_level_1', 'administrative_area_level_2'
];

// By default, only basic place data is requested (no extra billing)
const BASIC_DATA_FIELDS = [
  'address_components', 'adr_address', 'alt_id',
  'formatted_address', 'geometry', 'icon', 'id', 'name',
  'business_status', 'photo', 'place_id', 'scope', 'type',
  'url', 'utc_offset_minutes', 'vicinity'
];

export default {
  name: 'VueGoogleAutocomplete',

  props: {
    id: { type: String, required: true },
    classname: String,
    placeholder: { type: String, default: 'Start typing' },
    disabled: { type: Boolean, default: false },
    types: { type: String, default: 'address' },
    fields: { type: Array, default() { return BASIC_DATA_FIELDS; } },
    country: { type: [String, Array], default: null },
    enableGeolocation: { type: Boolean, default: false },
    geolocationOptions: {
      type: Object,
      default() { return { timeout: 5000, maximumAge: 600000, enableHighAccuracy: true }; }
    }
  },

  data() {
    return {
      autocompleteText: '',
      geolocation: { position: null, geocoder: null }
    };
  },

  watch: {
    // Emit input change events when text changes
    autocompleteText(newVal, oldVal) {
      this.$emit('inputChange', { newVal, oldVal }, this.id);
    },
    // Update component restrictions when country prop changes
    country(newVal) {
      if (this.autocomplete) {
        this.autocomplete.setComponentRestrictions({ country: newVal === null ? [] : newVal });
      }
    }
  },

  mounted: async function() {
    const options = {};
    if (this.types) options.types = [this.types];
    if (this.country) options.componentRestrictions = { country: this.country === null ? [] : this.country };
    if (this.fields && this.fields.length) options.fields = this.fields;

    // Load the new Places library and create the autocomplete element
    await google.maps.importLibrary('places');
    this.autocompleteEl = new google.maps.places.PlaceAutocompleteElement(options);
    this.autocomplete = this.autocompleteEl; // alias for backward compatibility
    this.$refs.autocompleteContainer.appendChild(this.autocompleteEl);

    // Listen for selection events and fetch full place details
    this.autocompleteEl.addEventListener('gmp-select', async (e) => {
      const place = e.placePrediction.toPlace();
      await place.fetchFields({ fields: this.fields && this.fields.length ? this.fields : BASIC_DATA_FIELDS });
      this.$emit('placechanged', place, e.placePrediction, this.id);
    });
  },

  methods: {
    // Triggered when input is focused
    onFocus() {
      this.biasAutocompleteLocation();
      this.$emit('focus');
    },
    // Triggered on blur
    onBlur() { this.$emit('blur'); },
    // Triggered on value change
    onChange() { this.$emit('change', this.autocompleteText); },
    onKeyPress(event) { this.$emit('keypress', event); },
    onKeyUp(event) { this.$emit('keyup', event); },

    // Reverse geocode given coordinates
    updateCoordinates(value) {
      if (!value || (!value.lat && !value.lng)) return;
      if (!this.geolocation.geocoder) {
        this.geolocation.geocoder = new google.maps.Geocoder();
      }
      this.geolocation.geocoder.geocode({ location: value }, (results, status) => {
        if (status === 'OK' && results[0]) {
          results = this.filterGeocodeResultTypes(results);
          const formatted = this.formatResult(results[0]);
          this.$emit('placechanged', formatted, results[0], this.id);
          this.update(results[0].formatted_address);
        } else {
          this.$emit('error', status === 'OK' ? 'no result for provided coordinates' : 'error getting address from coords');
        }
      });
    },

    // Use browser geolocation to bias predictions
    geolocate() {
      if (!navigator.geolocation) {
        this.$emit('error', 'Geolocation not supported');
        return;
      }
      navigator.geolocation.getCurrentPosition(
        (position) => {
          const geolocation = { lat: position.coords.latitude, lng: position.coords.longitude };
          this.updateCoordinates(geolocation);
          this.position = position;
        },
        (err) => {
          this.$emit('error', 'Cannot get Coordinates from navigator', err);
        },
        this.geolocationOptions
      );
    },

    // Bias autocomplete to user's current location
    biasAutocompleteLocation() {
      if (this.enableGeolocation && this.position) {
        const circle = new google.maps.Circle({ center: { lat: this.position.coords.latitude, lng: this.position.coords.longitude }, radius: this.position.coords.accuracy });
        this.autocomplete.setBounds(circle.getBounds());
      }
    },

    // Format a PlaceResult into a simpler object
    formatResult(place) {
      const returnData = {};
      for (let comp of place.address_components) {
        const key = comp.types[0];
        if (ADDRESS_COMPONENTS[key]) {
          returnData[key] = comp[ADDRESS_COMPONENTS[key]];
        }
      }
      return { formatted_address: place.formatted_address, ...returnData };
    },

    // Filter geocode results based on requested types
    filterGeocodeResultTypes(results) {
      if (!results || !this.types) return results;
      let output = [];
      let types = [this.types];
      if (types.includes('(cities)')) types = types.concat(CITIES_TYPE);
      if (types.includes('(regions)')) types = types.concat(REGIONS_TYPE);
      for (let r of results) {
        if (r.types.some(t => types.includes(t))) output.push(r);
      }
      return output;
    }
  }
};
</script>
