<script setup lang="ts">
import { getHeadings, splitContent } from '@scalar/code-highlight/markdown'
import { ScalarMarkdown } from '@scalar/components'
import GithubSlugger from 'github-slugger'
import { computed } from 'vue'

import { joinWithSlash } from '../../../helpers'
import { useNavState } from '../../../hooks'
import IntersectionObserver from '../../IntersectionObserver.vue'

const props = defineProps<{
  /** Markdown document */
  value?: string
}>()

/**
 * Descriptions, but split into multiple sections.
 * We need this to wrap the headings in IntersectionObserver components.
 */
const sections = computed(() => {
  if (!props.value) {
    return []
  }

  const slugger = new GithubSlugger()

  const items = splitContent(props.value).map((markdown) => {
    // Get “first” (and only) heading, if available
    const [heading] = getHeadings(markdown)

    // Generate an id for the heading
    const id = heading
      ? getHeadingId({
          ...heading,
          slug: slugger.slug(heading.value),
        })
      : undefined

    return {
      id,
      content: markdown,
    }
  })

  return items
})

const { getHeadingId, isIntersectionEnabled, replaceUrlState } = useNavState()

function handleScroll(headingId = '') {
  if (!isIntersectionEnabled.value) return

  replaceUrlState(headingId)
}

const slugger = new GithubSlugger()

/** Add ids to all headings */
const transformHeading = (node: Record<string, any>) => {
  node.data = {
    hProperties: {
      id: getHeadingId({
        depth: node.depth,
        value: node.children[0].value,
        slug: slugger.slug(node.children[0].value),
      }),
    },
  }

  return node
}
</script>

<template>
  <div
    v-if="value"
    class="introduction-description">
    <template
      v-for="section in sections"
      :key="section.id">
      <!-- headings -->
      <template v-if="section.id">
        <IntersectionObserver
          :id="section.id"
          class="introduction-description-heading"
          @intersecting="() => handleScroll(section.id)">
          <ScalarMarkdown
            :transform="transformHeading"
            transformType="heading"
            :value="section.content" />
        </IntersectionObserver>
      </template>
      <!-- everything else -->
      <template v-else>
        <ScalarMarkdown
          :value="section.content"
          withImages />
      </template>
    </template>
  </div>
</template>

<style scoped>
.introduction-description-heading {
  scroll-margin-top: 64px;
  margin-top: 1em;
  margin-bottom: 0.5em;
}
.markdown + .markdown {
  margin-top: 1em;
}
.introduction-description {
  display: flex;
  flex-direction: column;
}
.references-classic .introduction-description :deep(img) {
  max-width: 720px;
}
</style>
