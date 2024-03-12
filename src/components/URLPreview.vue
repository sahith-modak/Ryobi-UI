<template>
  <div>
    <div v-if="preview" class="link-preview">
      <h3><a :href="url" target="_blank">{{ preview.title }}</a></h3>
      <p>{{ preview.description }}</p>
      <img v-if="preview.image" :src="preview.image" alt="Preview Image" />
    </div>
    <div v-else>
      <h3><a :href="url" target="_blank">{{url}}</a></h3>
    </div>
  </div>
</template>

<script>
import { onMounted, ref } from 'vue';

export default {
  name: 'LinkPreview',
  props: {
    url: {
      type: String,
      required: true
    }
  },
  setup(props) {
    const preview = ref(null);

    const fetchPreview = async () => {
      try {
        debugger
        const response = await fetch(props.url);
        const html = await response.text();

        const parser = new DOMParser();
        const doc = parser.parseFromString(html, 'text/html');

        const title = doc.querySelector('title').textContent;
        const descriptionTag = doc.querySelector('meta[name="description"]');
        const description = descriptionTag ? descriptionTag.getAttribute('content') : '';
        const imageTag = doc.querySelector('meta[property="og:image"]');
        const image = imageTag ? imageTag.getAttribute('content') : '';

        preview.value = { title, description, image };
      } catch (error) {
        console.error('Error fetching link preview:', error);
      }
    };
    onMounted(() => fetchPreview())
    return {
      preview,
      fetchPreview
    };
  }
};
</script>

<style scoped>
.link-preview {
  border: 1px solid #ccc;
  padding: 10px;
}
.link-preview img {
  max-width: 100%;
}
</style>
