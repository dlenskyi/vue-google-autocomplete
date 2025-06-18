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
        {{
          sugg.placePrediction.text
            ? sugg.placePrediction.text.toString()
            : (sugg.placePrediction.description || '')
        }}
      </li>;;
await place.fetchFields({ fields: fieldsToRequest });;
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
