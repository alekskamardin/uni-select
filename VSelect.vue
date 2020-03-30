<template>
  <div v-click-outside="close" :style="containerStyles" class="input" @click="swapToInput">
    <div :style="styles" class="input__wrapper" :class="generalDynamicClass">
      <div v-if="selectedItem" class="input__selected" @click="innerInputClick">
        <slot :name="`item-${selectedItemSlotName}`" />
      </div>

      <div v-show="!selectedItem" class="input__stub">
        <img v-if="hasIcon" :src="iconSource" class="input__icon" />
        <input
          ref="input"
          :disabled="isDisabled"
          :placeholder="placeholder"
          :autocomplete="autofill"
          :value="value"
          :type="type"
          :style="inputStyles"
          class="input__field"
          @input="inputHandler"
          @focus="handleFocus"
          @blur="handleBlur"
          @keyup.enter="enterHandler"
          @change="emitEvents"
        />
        <slot name="icon" />
      </div>

      <ul v-show="hasAutocomplete && isSelectShowed" class="input__select" :style="styles">
        <div class="input__select-wrapper" :style="wrapperStyles">
          <li
            v-for="(item, index) in filteredItems"
            :key="index"
            class="input__select-item"
            @click="handleClick(item, $event)"
          >
            <slot :name="`item-${autocompleteItemSlotName(item)}`" />
          </li>

          <li v-if="!filteredItems.length" class="input__select-item input__select-item--error">
            Ничего не удалось найти :(
          </li>
        </div>
      </ul>

      <div
        v-if="hasAutocomplete && selectItems.length && !isDisabled"
        class="input__icon input__arrow"
        :class="rotatedArrow"
        @click="toggle"
      />
    </div>
    <div
      v-if="showLabelCondition"
      :style="labelStyle"
      class="input__label"
      :class="activeLabelClass"
      @click="focusInput"
    >
      {{ labelText }}
    </div>
  </div>
</template>

<script>
  import ClickOutside from 'vue-click-outside';

  export default {
    directives: {
      ClickOutside,
    },

    props: {
      type: { type: String, default: 'text' },
      value: { type: String, required: true },
      hasAutocomplete: { type: Boolean, default: false },
      autofill: { type: String, default: 'on' },
      autocompleteBy: { type: String, default: '' },
      defaultSelectedItem: { type: [String, Object], default: '' },
      isDisabled: { type: Boolean, default: false },
      placeholder: { type: String, default: '' },
      selectItems: { type: Array, default: () => [] },
      hasIcon: { type: Boolean, default: false },
      iconSource: { type: String, default: '' },
      bgColor: { type: String, default: 'white' },
      textColor: { type: String, default: 'white' },
      shadow: { type: String, default: 'none' },
      padding: { type: String, default: '4px 24px 4px 12px' },
      margin: { type: String, default: '0' },
      width: { type: String, default: '100%' },
      minWidth: { type: String, default: '100px' },
      maxWidth: { type: String, default: '416px' },
      height: { type: Number, default: 48 },
      fontSize: { type: Number, default: 12 },

      border: { type: String, default: 'solid 1px #acb6c3' },
      borderSuccess: { type: String, default: 'solid 1px rgba(2, 81, 248, 1)' },
      borderError: { type: String, default: 'solid 1px rgba(208, 2, 27, 1)' },
      borderRadius: { type: Number, default: 24 },

      filterFunction: { type: Function, default: null },
      validationFunction: { type: Function, default: null },

      labelText: { type: String, default: '' },

      labelSize: { type: Number, default: 14 },
      labelTop: { type: Number, default: 14 },
      labelLeft: { type: Number, default: 10 },

      labelActiveSize: { type: Number, default: 14 },
      labelActiveTopOffset: { type: Number, default: -24 },
      labelActiveLeftOffset: { type: Number, default: 22 },
    },

    data() {
      return {
        isSelectShowed: false,
        selectedItem: this.defaultSelectedItem,
        isValid: false,
        isError: false,
        searchResults: [],
        isSelectFocused: false,
        oldSelectedItem: '',
      };
    },

    computed: {
      filteredItems() {
        if (!this.searchResults.length && !this.value) return this.selectItems;
        if (this.searchResults && this.value) {
          return this.searchResults;
        }
        return [];
      },

      showLabelCondition() {
        if (!this.labelText) return false;
        if (this.isDisabled && this.value) {
          return false;
        }
        return true;
      },

      activeLabelClass() {
        return {
          'input__label--active': this.isLabelActive,
          'input__label--disabled': this.isDisabled,
        };
      },

      generalDynamicClass() {
        return {
          'input--disabled': this.isDisabled,
          'input--focused': this.isSelectFocused,
          'input__wrapper--straight': this.isSelectShowed,
          'input__wrapper--success': this.isValid || this.selectedItem,
          'input__wrapper--error': this.isError,
        };
      },

      withDefaultAutocomplete() {
        return !this.autocompleteBy;
      },

      selectedItemSlotName() {
        return this.withDefaultAutocomplete
          ? this.selectedItem
          : this.selectedItem[this.autocompleteBy];
      },

      rotatedArrow() {
        return { 'input__arrow--rotated': this.isSelectShowed };
      },

      calculatedListHeight() {
        return this.searchResults.length <= 6 ? 'fit-content' : '300px';
      },

      containerStyles() {
        return {
          '--width': this.width,
          '--max-width': this.maxWidth,
          '--min-width': this.minWidth,
          '--select-height': this.calculatedListHeight,
        };
      },

      wrapperStyles() {
        return {
          '--select-wrapper-height': `calc(${this.calculatedListHeight} - 15px)`,
        };
      },

      styles() {
        return {
          '--background-color': this.bgColor,
          '--text-color': this.textColor,
          '--padding': this.padding,
          '--margin': this.margin,
          '--width': this.width,
          '--height': `${this.height}px`,
          '--font-size': `${this.fontSize}px`,

          '--box-shadow': this.shadow,
          '--border': this.border,
          '--border-success': this.borderSuccess,
          '--border-error': this.borderError,
          '--border-radius': `${this.borderRadius}px`,
        };
      },

      inputStyles() {
        return {
          '--padding': this.padding,
          '--height': `${this.height - 4}px`,
          '--border-radius': `${this.borderRadius}px`,
        };
      },

      labelStyle() {
        return {
          '--label-size': `${this.labelSize}px`,
          '--label-top': `${this.labelTop}px`,
          '--label-left': `${this.labelLeft}px`,
          '--label-active-size': `${this.labelActiveSize}px`,
          '--label-active-top-offset': `${this.labelActiveTopOffset}px`,
          '--label-active-left-offset': `${this.labelActiveLeftOffset}px`,
        };
      },

      isLabelActive() {
        return this.value || this.selectedItem || this.isSelectFocused;
      },
    },

    methods: {
      enterHandler() {
        this.checkValidation();
        this.$emit('enterPressed', this.value);
      },

      innerInputClick(event) {
        event.stopPropagation();
        this.swapToInput();
      },

      handleBlur(event) {
        this.isSelectFocused = false;
        this.checkValidation();
        this.$emit('inputBlur', event);
      },

      checkValidation() {
        if (this.validationFunction && this.value) {
          if (this.validationFunction(this.value)) {
            this.isValid = true;
            this.isError = false;
          } else {
            this.isValid = false;
            this.isError = true;
          }
        } else {
          this.isValid = false;
          this.isError = false;
        }
      },

      focusInput() {
        this.$nextTick(() => this.$refs.input.focus());
      },

      handleFocus(event) {
        if (this.hasAutocomplete) {
          this.isSelectShowed = true;
        }
        this.isSelectFocused = true;
        this.$emit('inputFocus', event);
      },

      inputHandler(event) {
        const {
          target: { value },
        } = event;

        if (this.validationFunction) {
          this.checkValidation();
        }

        const normalizedValue = value.toLowerCase();
        this.$emit('input', event.target.value);

        if (!this.hasAutocomplete) return;

        if (!this.filterFunction) {
          if (!this.withDefaultAutocomplete) return;
          this.searchResults = this.selectItems.filter(item =>
            item.toLowerCase().includes(normalizedValue)
          );
        } else {
          this.searchResults = this.filterFunction(this.selectItems, normalizedValue);
        }

        if (this.value) {
          this.isSelectShowed = true;
        }

        if (!event.target.value) {
          this.searchResults = [];
        }
      },

      autocompleteItemSlotName(item) {
        return this.withDefaultAutocomplete ? item : item[this.autocompleteBy];
      },

      emitEvents(event) {
        this.$emit('change', this.inputValue);
        this.$emit('changeEvent', event);
      },

      handleClick(item, event) {
        event.stopPropagation();
        this.selectedItem = item;
        this.$emit('input', '');
        this.$emit('itemClick', item);
        this.isValid = true;
        this.searchResults = [];
        this.isSelectShowed = false;
      },

      toggle() {
        this.isSelectShowed = !this.isSelectShowed;
      },

      close() {
        if (!this.selectedItem && !this.value) {
          this.selectedItem = this.oldSelectedItem;
        }
        this.isSelectShowed = false;
      },

      swapToInput() {
        // eslint-disable-next-line
        console.log(this.selectedItem);
        if (this.isDisabled) return;
        if (this.selectedItem) {
          this.oldSelectedItem = this.selectedItem;
          this.selectedItem = '';
          this.$emit('input', '');
          this.$emit('reset');
          this.isValid = false;
          this.searchResults = [];
          this.$nextTick(() => this.$refs.input.focus());
        }
      },
    },
  };
</script>

<style lang="scss" scoped>
  @import '../assets/variables.scss';

  .input {
    display: flex;
    position: relative;
    width: var(--width);
    min-width: var(--min-width);
    max-width: var(--max-width);

    &--focused {
      border: solid 2px $blue !important;
    }

    &__icon {
      width: 24px;
      height: 24px;
    }

    &__remove {
      cursor: pointer;
      transition: opacity 0.3s;

      &:hover {
        opacity: 0.7;
      }
    }

    &__stub {
      width: 100%;
      height: 100%;
      display: flex;
      position: absolute;
      top: 0;
      left: 0;
      background: transparent;
    }

    &__selected {
      width: 100%;
      color: $black;
    }

    &__wrapper {
      background-color: var(--background-color);
      width: 100%;
      height: var(--height);
      color: var(--text-color);
      font-size: var(--font-size);
      border: var(--border);
      border-radius: var(--border-radius);
      box-shadow: var(--box-shadow);
      padding: var(--padding);
      margin: var(--margin);

      display: flex;
      box-sizing: border-box;
      align-items: center;
      justify-content: space-between;

      &--straight {
        border-bottom-left-radius: 0;
        border-bottom-right-radius: 0;
      }

      &--success {
        border: var(--border-success);
      }

      &--error {
        border: var(--border-error);
      }
    }

    &__select {
      position: absolute;
      top: 48px;
      display: flex;
      flex-direction: column;
      width: 100%;
      left: 0;
      margin: 0;
      padding: 0;
      list-style: none;
      color: $black;
      background-color: var(--background-color);
      box-sizing: border-box;
      border: var(--border);
      box-shadow: var(--box-shadow);
      border-top: none;
      border-bottom-left-radius: var(--border-radius);
      border-bottom-right-radius: var(--border-radius);
      height: var(--select-height);
      max-height: 300px;
      overflow-y: hidden;
      z-index: 1;
    }

    &__select-wrapper {
      display: flex;
      flex-direction: column;
      max-height: var(--select-wrapper-height);
      overflow-y: auto;
      padding-bottom: 4px;
    }

    &__select-item {
      display: flex;
      align-items: center;
      min-height: 48px;
      box-sizing: border-box;
      padding: 0 12px;
      cursor: pointer;
      transition: box-shadow 0.3s;

      &:hover {
        box-shadow: 0 -0.1px 3px 0 rgba($shadow-color, 0.08), 0 4px 4px 0 rgba($shadow-color, 0.1),
          0 1px 5px 0 rgba($shadow-color, 0.15);
      }

      > div {
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
      }

      &--error {
        cursor: default;
        &:hover {
          box-shadow: none;
        }
      }
    }

    &__label {
      position: absolute;
      background: $white;
      color: rgba($black, 0.4);
      top: var(--label-top);
      left: var(--label-left);
      font-size: var(--label-size);
      padding: 2px;
      transition: transform 0.3s;
      width: calc(100% - 50px);

      &--active {
        transform: translate(var(--label-active-left-offset), var(--label-active-top-offset));
        width: unset;
      }

      &--disabled {
        background-color: transparent;
      }
    }

    &__field {
      border: none;
      outline: none;
      width: 100%;
      box-sizing: border-box;
      background: transparent;
      padding: var(--padding);
      height: var(--height);
      border-radius: var(--border-radius);
      margin: 2px;
      box-shadow: 0 0 0 1000px white inset !important;

      &:disabled {
        background: rgba($grey1, 0.1);
        box-shadow: none !important;
      }
    }

    &__arrow {
      position: absolute;
      right: 10px;
      min-width: 20px;
      min-height: 20px;
      transition: all 0.3s;
      background: url('../assets/icon-arrow.svg');
      background-position: center;
      background-repeat: no-repeat;
      background-size: contain;
      cursor: pointer;

      &:hover {
        opacity: 0.7;
      }

      &--rotated {
        transform: rotate(-180deg);
      }
    }
  }
</style>
