<template>
  <div
    :class="`magnifier ${className}`"
    :style="mgWrapperStyle"
  >
    <img
      ref="img"
      :src="src"
      v-bind="$attrs"
      :style="mgImgStyle"
      @load="onImageLoad"
      @mouseenter="onMouseEnter"
      @mousemove="throttleOnMouseMove"
      @mouseout="onMouseOut"
      @touchstart.prevent="onTouchStart"
      @touchmove.prevent="throttleOnTouchMove"
      @touchend="onTouchEnd"
    />

    <Teleport v-if="showZoom" to="body">
      <div
        v-if="imgBounds && mgShow"
        :class="mgClasses"
        :style="mgStyle"
      />
    </Teleport>
  </div>
</template>

<script>
import { computed, onMounted, onUnmounted, ref } from 'vue';
import {debounce, throttle} from 'lodash';
// import  from 'lodash';

/**
 * @module Magnifier
 * @description This component provides a magnifying glass effect over an image. It supports various customization options such as zoom factor, magnifying glass dimensions, and offsets for mouse or touch interactions.
 */

/**
 * @typedef {Object} Props
 * @property {string} src - URL/path of the large image. This prop is required.
 * @property {number|string} [height='auto'] - Image height. Can be specified in absolute (px) or relative (%) values.
 * @property {number|string} [width='100%'] - Image width. Can be specified in absolute (px) or relative (%) values.
 * @property {string} [className=''] - Class which will be applied to the image wrapper.
 * @property {string} [zoomImgSrc=''] - URL/path of the image inside the magnifying glass. If not specified, the large image will be used.
 * @property {number} [zoomFactor=1.5] - Factor by which the zoom image will be scaled, based on the size of the large image.
 * @property {number} [mgWidth=150] - Width of the magnifying glass in pixels.
 * @property {number} [mgHeight=150] - Height of the magnifying glass in pixels.
 * @property {number} [mgBorderWidth=2] - Border width of the magnifying glass in pixels.
 * @property {string} [mgShape='circle'] - Shape of the magnifying glass. Possible values: 'circle', 'square'.
 * @property {boolean} [mgShowOverflow=true] - Set this to false to cut off the magnifying glass at the image borders. When disabling mgShowOverflow, it's recommended that you also set all offsets to 0.
 * @property {number} [mgMouseOffsetX=0] - Horizontal offset of the magnifying glass in pixels when hovering with a mouse.
 * @property {number} [mgMouseOffsetY=0] - Vertical offset of the magnifying glass in pixels when hovering with a mouse.
 * @property {number} [mgTouchOffsetX=-50] - Horizontal offset of the magnifying glass in pixels when dragging on a touch screen.
 * @property {number} [mgTouchOffsetY=-50] - Vertical offset of the magnifying glass in pixels when dragging on a touch screen.
 * @property {boolean} [mgShow=true] - Show or hide the magnifying glass.
 * @property {string} [mgCornerBgColor='#fff'] - Background color of the magnifying glass corner.
 */

/**
 * @typedef {Object} Events
 * @property {Event} image:load - This event is emitted when the image has fully loaded.
 */

export default {
  inheritAttrs: false,
  props: {
    // Image
    src: { type: String, required: true },
    width: { type: [String, Number], default: '100%' },
    height: { type: [String, Number], default: 'auto' },
    className: { type: String, default: '' },

    // Zoom image
    zoomImgSrc: { type: String, default: '' },
    zoomFactor: { type: Number, default: 1.5 },

    // Magnifying glass
    mgWidth: { type: Number, default: 150 },
    mgHeight: { type: Number, default: 150 },
    mgBorderWidth: { type: Number, default: 2 },
    mgShape: { type: String, default: 'circle' },
    mgShowOverflow: { type: Boolean, default: true },
    mgMouseOffsetX: { type: Number, default: 0 },
    mgMouseOffsetY: { type: Number, default: 0 },
    mgTouchOffsetX: { type: Number, default: -50 },
    mgTouchOffsetY: { type: Number, default: -50 },
    mgShow: { type: Boolean, default: true },
    mgCornerBgColor: { type: String, default: '#fff' }
  },

  emits: ['image:load'],

  setup(props, { emit }) {
    const img = ref(null);
    const imgBounds = ref(null);
    const showZoom = ref(false);
    const mgOffsetX = ref(0);
    const mgOffsetY = ref(0);
    const relX = ref(0);
    const relY = ref(0);

    const mgClasses = computed(() => {
      let classes = 'magnifier__glass';
      if (showZoom.value) classes += ' magnifier__visible';
      if (props.mgShape === 'circle') classes += ' magnifier__circle';
      return classes;
    });

    const mgWrapperStyle = computed(() => ({
      width: props.width,
      height: props.height,
      overflow: props.mgShowOverflow ? 'visible' : 'hidden'
    }));

    const mgImgStyle = computed(() => ({
      cursor: props.mgShow ? 'none' : '',
      width: '100%',
      height: '100%'
    }));

    const mgStyle = computed(() => {
      if (!imgBounds.value) return {};

      const viewportWidth = window.innerWidth;

      const x = relX.value * imgBounds.value.width - props.mgWidth / 2 + mgOffsetX.value;
      const y = relY.value * imgBounds.value.height - props.mgHeight / 2 + mgOffsetY.value;
      const top = imgBounds.value.top + y + window.scrollY;

      let left = imgBounds.value.left + x + window.scrollX;

      if (left < window.scrollX) left = window.scrollX;
      if (left + props.mgWidth > viewportWidth + window.scrollX) {
        left = viewportWidth + window.scrollX - props.mgWidth;
      }

      return {
        width: `${props.mgWidth}px`,
        height: `${props.mgHeight}px`,
        left: `${left}px`,
        top: `${top}px`,
        backgroundImage: `url(${props.zoomImgSrc || props.src})`,
        backgroundPosition: `calc(${relX.value * 100}% + ${
          props.mgWidth / 2
        }px - ${relX.value * props.mgWidth}px) calc(${relY.value * 100}% + ${
          props.mgHeight / 2
        }px - ${relY.value * props.mgWidth}px)`,
        backgroundSize: `${props.zoomFactor * imgBounds.value.width}% ${props.zoomFactor * imgBounds.value.height}%`,
        borderWidth: `${props.mgBorderWidth}px`,
        backgroundColor: props.mgCornerBgColor
      };
    });

    const calcImgBounds = () => {
      console.log('calcImgBounds', img.value);
      
      if (img.value) imgBounds.value = img.value.getBoundingClientRect();
    };

    const onImageLoad = (event) => {
      emit('image:load', event);
      calcImgBounds();
    };

    const onMouseEnter = () => {
      calcImgBounds();
    };

    const onMouseMove = (e) => {
      if (!imgBounds.value) return;

      const target = e.target;

      mgOffsetX.value = props.mgMouseOffsetX;
      mgOffsetY.value = props.mgMouseOffsetY;

      relX.value = (e.clientX - imgBounds.value.left) / target.clientWidth;
      relY.value = (e.clientY - imgBounds.value.top) / target.clientHeight;

      showZoom.value = true;
    };

    const onMouseOut = () => {
      showZoom.value = false;
    };

    const onTouchStart = () => {
      calcImgBounds();
    };

    const onTouchMove = (e) => {
      if (!imgBounds.value) return;

      const target = e.target;

      const relXLocal = (e.targetTouches[0].clientX - imgBounds.value.left) / target.clientWidth;
      const relYLocal = (e.targetTouches[0].clientY - imgBounds.value.top) / target.clientHeight;

      if (relXLocal >= 0 && relYLocal >= 0 && relXLocal <= 1 && relYLocal <= 1) {
        mgOffsetX.value = props.mgTouchOffsetX;
        mgOffsetY.value = props.mgTouchOffsetY;

        relX.value = relXLocal;
        relY.value = relYLocal;

        showZoom.value = true;
      } else {
        showZoom.value = false;
      }
    };

    const onTouchEnd = () => {
      showZoom.value = false;
    };

    const calcImgBoundsDebounced = debounce(calcImgBounds, 200);

    const fps = 1000 / 120;
    const throttleOnMouseMove = throttle(onMouseMove, fps, { trailing: false });
    const throttleOnTouchMove = throttle(onTouchMove, fps, { trailing: false });

    if (import.meta.client) {
      onMounted(() => {
        window.addEventListener('resize', calcImgBoundsDebounced);
        window.addEventListener('scroll', calcImgBoundsDebounced, true);
      });

      onUnmounted(() => {
        window.removeEventListener('resize', calcImgBoundsDebounced);
        window.removeEventListener('scroll', calcImgBoundsDebounced, true);
      });
    }

    return {
      img,
      imgBounds,
      showZoom,
      mgOffsetX,
      mgOffsetY,
      relX,
      relY,
      mgClasses,
      mgWrapperStyle,
      mgImgStyle,
      mgStyle,
      onImageLoad,
      onMouseEnter,
      throttleOnMouseMove,
      onMouseOut,
      onTouchStart,
      throttleOnTouchMove,
      onTouchEnd
    };
  }
};
</script>

<style lang="scss" scoped>
.magnifier__glass {
  position: absolute;
  z-index: 1000;
  background: #fff no-repeat;
  border: solid #ebebeb;
  box-shadow: 2px 2px 3px rgba(0, 0, 0, 0.3);
  opacity: 0;
  transition: opacity 0.3s;
  pointer-events: none;
}

.magnifier__glass.magnifier__circle {
  border-radius: 50%;
}

.magnifier__glass.magnifier__visible {
  opacity: 1;
}
</style>
