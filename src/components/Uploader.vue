<template>
  <div class="csv-uploader">
    <input type="file" @change="handleFileUpload" accept=".csv" />
  </div>
</template>

<script>
export default {
  props: ["handleFileUpload"],
};
</script>
