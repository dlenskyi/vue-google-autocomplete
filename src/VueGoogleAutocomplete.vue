<template>
  <div ref="container"></div>
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

const CITIES_TYPE = ['locality', 'administrative_area_level_3'];
const REGIONS_TYPE = [
  'locality','sublocality','postal_code','country',
  'administrative_area_level_1','administrative_area_level_2'
];

const BASIC_DATA_FIELDS = [
  'address_components','formatted_address','geometry','name',
  'place_id'
];

export default {
  name: 'VueGoogleAutocomplete',

  props: {
    id:               { type: String,   required: true },
    classname:        String,
    placeholder:      { type: String,   default: 'Start typing' },
    disabled:         { type: Boolean,  default: false },
    types:            { type: String,   default: 'address' },
    fields:           { type: Array,    default: () => BASIC_DATA_FIELDS },
    country:          { type: [String,Array], default: null },
    enableGeolocation:{ type: Boolean,  default: false },
    geolocationOptions:{ type: Object,  default: null },
    apiKey:           { type: String,   required: true },
    language:         { type: String,   default: 'en' },
  },

  data() {
    return {
      autocompleteText: '',
      geolocation: { geocoder: null, position: null }
    };
  },

  async mounted() {
    await new Promise(resolve => {
      if (window.google && google.maps.places) resolve();
      window.initGoogleMaps = resolve;
    });
  
    this.placeAutocomplete = new google.maps.places.PlaceAutocompleteElement({
      componentRestrictions: this.country ? { country: this.country } : undefined,
      types: this.types ? [this.types] : undefined,
      // locationRestriction: map.getBounds(),
    });
  
    this.placeAutocomplete.id = this.id;
  
    this.$refs.container.appendChild(this.placeAutocomplete);
  
    this.placeAutocomplete.addEventListener('gmp-select', async (event) => {
      const place = event.placePrediction.toPlace();
      await place.fetchFields({ fields: this.fields });
      const result = this.formatResult(place);
      this.$emit('placechanged', result, place, this.id);
      this.autocompleteText = place.displayName || place.formattedAddress;
      this.$emit('change', this.autocompleteText);
    });
  },

  methods: {
    _loadBetaMapsAPI() {
      if (window.google && google.maps && google.maps.places) {
        return Promise.resolve();
      }
      return new Promise((resolve, reject) => {
        window.initGMaps = () => resolve();
        const s = document.createElement('script');
        s.async = true;
        s.defer = true;
        s.src = [
          'https://maps.googleapis.com/maps/api/js',
          `?key=${this.apiKey}`,
          `&libraries=places`,
          `&language=${this.language}`,
          `&v=beta`,
          `&loading=async`,
          `&callback=initGMaps`
        ].join('');
        s.onerror = reject;
        document.head.appendChild(s);
      });
    },

    _initElement() {
      // создаём новый виджет
      const el = new google.maps.places.PlaceAutocompleteElement();
      if (this.id)          el.id            = this.id;
      if (this.classname)   el.className     = this.classname;
      if (this.placeholder) el.placeholder   = this.placeholder;
      if (this.disabled)    el.disabled      = true;

      // ограничения
      el.includedPrimaryTypes = this.types ? [this.types] : [];
      el.includedRegionCodes  = this.country
        ? (Array.isArray(this.country) ? this.country : [this.country])
        : [];

      this.$refs.container.appendChild(el);
      this._element = el;

      // событие выбора
      el.addEventListener('gmp-select', async ({ placePrediction }) => {
        const place = placePrediction.toPlace();
        await place.fetchFields({ fields: this.fields });
        this.autocompleteText = el.value;
        this.$emit('placechanged', this.formatResult(place), place, this.id);
        this.$emit('change', this.autocompleteText);
      });

      // focus / blur / key события
      el.addEventListener('focus', () => this.$emit('focus'));
      el.addEventListener('blur',  () => this.$emit('blur'));
      el.addEventListener('input', e => {
        const newVal = e.target.value;
        const oldVal = this.autocompleteText;
        this.autocompleteText = newVal;
        this.$emit('inputChange',{newVal,oldVal},this.id);
      });
      el.addEventListener('keypress', e => this.$emit('keypress', e));
      el.addEventListener('keyup',     e => this.$emit('keyup', e));
    },

    formatResult(place) {
      const data = {};
      for (const comp of place.address_components || []) {
        const key = comp.types[0];
        if (ADDRESS_COMPONENTS[key]) {
          data[key] = comp[ADDRESS_COMPONENTS[key]];
        }
      }
      data.latitude  = place.geometry.location.lat();
      data.longitude = place.geometry.location.lng();
      data.formatted_address = place.formatted_address;
      return data;
    },

    // обратное гео по координатам
    updateCoordinates(value) {
      if (!value || value.lat==null || value.lng==null) return;
      if (!this.geolocation.geocoder) {
        this.geolocation.geocoder = new google.maps.Geocoder();
      }
      this.geolocation.geocoder.geocode({ location: value }, (res, status) => {
        if (status==='OK' && res[0]) {
          const first = this.filterGeocodeResultTypes(res)[0];
          this.$emit('placechanged', this.formatResult(first), first, this.id);
          this._element.value = first.formatted_address;
        } else {
          this.$emit('error', status);
        }
      });
    },

    geolocate() {
      if (!navigator.geolocation) {
        return this.$emit('error','Geolocation not supported');
      }
      navigator.geolocation.getCurrentPosition(
        pos => this.updateCoordinates({
          lat: pos.coords.latitude,
          lng: pos.coords.longitude
        }),
        err => this.$emit('error', err),
        this.geolocationOptions
      );
    },

    clear() {
      this._element.value = '';
    },
    focus() {
      this._element.focus();
    },
    blur() {
      this._element.blur();
    },

    filterGeocodeResultTypes(results) {
      if (!results || !this.types) return results;
      let types = [this.types];
      if (types.includes('(cities)'))  types = types.concat(CITIES_TYPE);
      if (types.includes('(regions)')) types = types.concat(REGIONS_TYPE);
      return results.filter(r => r.types.some(t => types.includes(t)));
    }
  }
};
</script>
