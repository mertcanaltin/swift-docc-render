<!--
  This source file is part of the Swift.org open source project

  Copyright (c) 2021-2023 Apple Inc. and the Swift project authors
  Licensed under Apache License v2.0 with Runtime Library Exception

  See https://swift.org/LICENSE.txt for license information
  See https://swift.org/CONTRIBUTORS.txt for Swift project authors
-->

<template>
  <ConditionalWrapper
    ref="wrapper"
    :tag="DeviceFrameComponent"
    :should-wrap="!!deviceFrame"
    :device="deviceFrame"
  >
    <video
      ref="video"
      :controls="showsControls"
      :autoplay="autoplays"
      :poster="normalisedPosterPath"
      :muted="muted"
      :width="optimalWidth"
      playsinline
      @playing="$emit('playing')"
      @pause="$emit('pause')"
      @ended="$emit('ended')"
    >
      <!--
        Many browsers do not support the `media` attribute for `<source>` tags
        within a video specifically, so this implementation for dark theme assets
        is handled with JavaScript media query listeners unlike the `<source>`
        based implementation being used for image assets.
      -->
      <source :src="normalizePath(videoAttributes.url)">
    </video>
  </ConditionalWrapper>
</template>

<script>
import {
  separateVariantsByAppearance,
  normalizePath,
  getIntrinsicDimensions,
  extractDensities,
} from 'docc-render/utils/assets';
import AppStore from 'docc-render/stores/AppStore';
import ColorScheme from 'docc-render/constants/ColorScheme';
import ConditionalWrapper from 'docc-render/components/ConditionalWrapper.vue';
import DeviceFrame from 'docc-render/components/ContentNode/DeviceFrame.vue';

export default {
  name: 'VideoAsset',
  components: { ConditionalWrapper },
  props: {
    variants: {
      type: Array,
      required: true,
    },
    showsControls: {
      type: Boolean,
      default: () => true,
    },
    autoplays: {
      type: Boolean,
      default: () => true,
    },
    posterVariants: {
      type: Array,
      required: false,
      default: () => [],
    },
    muted: {
      type: Boolean,
      default: true,
    },
    deviceFrame: {
      type: String,
      required: false,
    },
  },
  data: () => ({
    appState: AppStore.state,
    optimalWidth: null,
  }),
  computed: {
    DeviceFrameComponent: () => DeviceFrame,
    preferredColorScheme: ({ appState }) => appState.preferredColorScheme,
    systemColorScheme: ({ appState }) => appState.systemColorScheme,
    userPrefersDark: ({
      preferredColorScheme,
      systemColorScheme,
    }) => preferredColorScheme === ColorScheme.dark || (
      preferredColorScheme === ColorScheme.auto
      && systemColorScheme === ColorScheme.dark
    ),
    shouldShowDarkVariant: ({
      darkVideoVariantAttributes,
      userPrefersDark,
    }) => darkVideoVariantAttributes && userPrefersDark,
    /**
     * Returns the light video if available, fallback to dark variant
     * @returns {Object}
     */
    defaultVideoAttributes() {
      return this.videoVariantsGroupedByAppearance.light[0]
        || this.darkVideoVariantAttributes
        || {};
    },
    darkVideoVariantAttributes() {
      return this.videoVariantsGroupedByAppearance.dark[0];
    },
    /**
     * Video variants for each style
     * @returns {{ light: [], dark: [] }}
     */
    videoVariantsGroupedByAppearance() {
      return separateVariantsByAppearance(this.variants);
    },
    /**
     * Return grouped poster variants per style
     * @returns {{ light: [], dark: [] }}
     */
    posterVariantsGroupedByAppearance() {
      const { light, dark } = separateVariantsByAppearance(this.posterVariants);
      return {
        light: extractDensities(light),
        dark: extractDensities(dark),
      };
    },
    /**
     * Returns dark poster variant if available and applicable
     * Returns light variant or empty object otherwise.
     * @returns {Object}
     */
    defaultPosterAttributes: ({
      posterVariantsGroupedByAppearance: variants,
      userPrefersDark,
    }) => (userPrefersDark && variants.dark.length
      ? variants.dark[0]
      : variants.light[0] || {}
    ),
    normalisedPosterPath: ({ defaultPosterAttributes }) => (
      normalizePath(defaultPosterAttributes.src)
    ),
    videoAttributes: ({
      darkVideoVariantAttributes,
      defaultVideoAttributes,
      shouldShowDarkVariant,
    }) => (shouldShowDarkVariant
      ? darkVideoVariantAttributes
      : defaultVideoAttributes
    ),
  },
  watch: {
    normalisedPosterPath: {
      immediate: true,
      handler: 'getPosterDimensions',
    },
  },
  methods: {
    normalizePath,
    async getPosterDimensions(path) {
      if (!path) {
        this.optimalWidth = null;
        return;
      }
      const { density } = this.defaultPosterAttributes;
      const currentVariantDensity = parseInt(density.match(/\d+/)[0], 10);
      const { width } = await getIntrinsicDimensions(path);
      this.optimalWidth = width / currentVariantDensity;
    },
  },
};
</script>
