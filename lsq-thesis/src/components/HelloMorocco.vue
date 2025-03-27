<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
    <h2>A Case Study of Morocco and Climate Change</h2>
  </div>
</template>

<script>
export default {
  name: 'HelloMorocco',
  props: {
    msg: String
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
html, body {
  width: 100%;
  margin: 0;
}
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
