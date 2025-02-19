<template>
  <div class="color-picker">
    <div class="picker-saturation-wrap">
      <Saturation :value="color" :hue="hue" @colorChange="value => changeColor(value)" />
    </div>
    <div class="picker-controls">
      <div class="picker-color-wrap">
        <div class="picker-current-color" :style="{ background: currentColor }"></div>
        <Checkboard />
      </div>
      <div class="picker-sliders">
        <div class="picker-hue-wrap">
          <Hue :value="color" :hue="hue" @colorChange="value => changeColor(value)" />
        </div>
        <div class="picker-alpha-wrap">
          <Alpha :value="color" @colorChange="value => changeColor(value)" />
        </div>
      </div>
    </div>

    <div class="picker-field">
      <EditableInput :value="color" @colorChange="value => changeColor(value)" />
    </div>

    <div class="picker-presets">
      <div
        class="picker-presets-color"
        v-for="c in themeColors"
        :key="c"
        :style="{background: c}"
        @click="selectPresetColor(c)"
      ></div>
    </div>

    <div class="picker-gradient-presets">
      <div
        class="picker-gradient-col"
        v-for="(col, index) in presetColors"
        :key="index"
      >
        <div class="picker-gradient-color"
          v-for="c in col"
          :key="c"
          :style="{background: c}"
          @click="selectPresetColor(c)"
        ></div>
      </div>
    </div>

    <div class="picker-presets">
      <div
        v-for="c in standardColors"
        :key="c"
        class="picker-presets-color"
        :style="{ background: c }"
        @click="selectPresetColor(c)"
      ></div>
    </div>

    <div class="recent-colors-title" v-if="recentColors.length">最近使用：</div>
    <div class="recent-colors">
      <div
        v-for="c in recentColors"
        :key="c"
        class="picker-presets-color"
        :style="{ background: c }"
        @click="selectPresetColor(c)"
      ></div>
    </div>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, onMounted, ref, watch } from 'vue'
import tinycolor, { ColorFormats } from 'tinycolor2'
import { debounce } from 'lodash'

import Alpha from './Alpha.vue'
import Checkboard from './Checkboard.vue'
import Hue from './Hue.vue'
import Saturation from './Saturation.vue'
import EditableInput from './EditableInput.vue'

const RECENT_COLORS = 'RECENT_COLORS'

const presetColorConfig = [
  ['#7f7f7f', '#f2f2f2'],
  ['#0d0d0d', '#808080'],
  ['#1c1a10', '#ddd8c3'],
  ['#0e243d', '#c6d9f0'],
  ['#233f5e', '#dae5f0'],
  ['#632623', '#f2dbdb'],
  ['#4d602c', '#eaf1de'],
  ['#3f3150', '#e6e0ec'],
  ['#1e5867', '#d9eef3'],
  ['#99490f', '#fee9da'],
]

const gradient = (startColor: string, endColor: string, step: number) => {
  const _startColor = tinycolor(startColor).toRgb()
  const _endColor = tinycolor(endColor).toRgb()

  const rStep = (_endColor.r - _startColor.r) / step
  const gStep = (_endColor.g - _startColor.g) / step
  const bStep = (_endColor.b - _startColor.b) / step
  const gradientColorArr = []

  for (let i = 0; i < step; i++) {
    const gradientColor = tinycolor({
      r: _startColor.r + rStep * i,
      g: _startColor.g + gStep * i,
      b: _startColor.b + bStep * i,
    }).toRgbString()
    gradientColorArr.push(gradientColor)
  }
  return gradientColorArr
}

const getPresetColors = () => {
  const presetColors = []
  for (const color of presetColorConfig) {
    presetColors.push(gradient(color[1], color[0], 5))
  }
  return presetColors
}

export default defineComponent({
  name: 'color-picker',
  components: {
    Alpha,
    Checkboard,
    Hue,
    Saturation,
    EditableInput,
  },
  props: {
    modelValue: {
      type: String,
      default: '#e86b99',
    },
  },
  setup(props, { emit }) {
    const hue = ref(-1)
    const recentColors = ref<string[]>([])

    const color = computed({
      get() {
        return tinycolor(props.modelValue).toRgb()
      },
      set(rgba: ColorFormats.RGBA) {
        const rgbaString = `rgba(${[rgba.r, rgba.g, rgba.b, rgba.a].join(',')})`
        emit('update:modelValue', rgbaString)
      },
    })

    const themeColors = ['#000000', '#ffffff', '#eeece1', '#1e497b', '#4e81bb', '#e2534d', '#9aba60', '#8165a0', '#47acc5', '#f9974c']
    const standardColors = ['#c21401', '#ff1e02', '#ffc12a', '#ffff3a', '#90cf5b', '#00af57', '#00afee', '#0071be', '#00215f', '#72349d']
    const presetColors = getPresetColors()

    const currentColor = computed(() => {
      return `rgba(${[color.value.r, color.value.g, color.value.b, color.value.a].join(',')})`
    })

    const selectPresetColor = (colorString: string) => {
      hue.value = tinycolor(colorString).toHsl().h
      emit('update:modelValue', colorString)
    }

    // 每次选择非预设颜色时，需要将该颜色加入到最近使用列表中
    const updateRecentColorsCache = debounce(function() {
      const _color = tinycolor(color.value).toRgbString()
      if (!recentColors.value.includes(_color)) {
        recentColors.value = [_color, ...recentColors.value]

        const maxLength = 10
        if (recentColors.value.length > maxLength) {
          recentColors.value = recentColors.value.slice(0, maxLength)
        }
      }
    }, 300, { trailing: true })

    onMounted(() => {
      const recentColorsCache = localStorage.getItem(RECENT_COLORS)
      if (recentColorsCache) recentColors.value = JSON.parse(recentColorsCache)
    })

    watch(recentColors, () => {
      const recentColorsCache = JSON.stringify(recentColors.value)
      localStorage.setItem(RECENT_COLORS, recentColorsCache)
    })

    const changeColor = (value: ColorFormats.RGBA | ColorFormats.HSLA | ColorFormats.HSVA) => {
      if ('h' in value) {
        hue.value = value.h
        color.value = tinycolor(value).toRgb()
      }
      else {
        hue.value = tinycolor(value).toHsl().h
        color.value = value
      }

      updateRecentColorsCache()
    }

    return {
      themeColors,
      standardColors,
      presetColors,
      color,
      hue,
      currentColor,
      changeColor,
      selectPresetColor,
      recentColors,
    }
  },
})
</script>

<style lang="scss" scoped>
.color-picker {
  position: relative;
  width: 240px;
  background: #fff;
  user-select: none;
  margin-bottom: -10px;
}
.picker-saturation-wrap {
  width: 100%;
  padding-bottom: 50%;
  position: relative;
  overflow: hidden;
}
.picker-controls {
  display: flex;
}
.picker-sliders {
  padding: 4px 0;
  flex: 1;
}
.picker-hue-wrap {
  position: relative;
  height: 10px;
}
.picker-alpha-wrap {
  position: relative;
  height: 10px;
  margin-top: 4px;
  overflow: hidden;
}
.picker-color-wrap {
  width: 24px;
  height: 24px;
  position: relative;
  margin-top: 4px;
  margin-right: 4px;
  outline: 1px dashed rgba($color: #666, $alpha: .12);

  .checkerboard {
    background-size: auto;
  }
}
.picker-current-color {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 2;
}

.picker-field {
  margin-bottom: 8px;
}

.picker-presets {
  @include flex-grid-layout();
}
.picker-presets-color {
  @include flex-grid-layout-children(10, 7%);

  height: 0;
  padding-bottom: 7%;
  flex-shrink: 0;
  position: relative;
  cursor: pointer;
}
.picker-gradient-presets {
  @include flex-grid-layout();
}
.picker-gradient-col {
  @include flex-grid-layout-children(10, 7%);

  display: flex;
  flex-direction: column;
}
.picker-gradient-color {
  width: 100%;
  height: 0;
  padding-bottom: 100%;
  position: relative;
  cursor: pointer;
}

.recent-colors-title {
  font-size: 12px;
  margin-bottom: 4px;
}
.recent-colors {
  @include flex-grid-layout();
}
</style>