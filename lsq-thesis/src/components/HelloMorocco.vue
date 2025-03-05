<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
    <h2>Extreme Weather Events and Climate Change</h2>
    <p>
      To check out other projects Lisa has created, visit
      <a href="https://lsq.design" target="_blank" rel="noopener">lsq.design</a>.
    </p>
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
