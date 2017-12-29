<template>
  <div id="app">
    <img src="./assets/logo.png">
    <br>
    <input v-model="q">
    <button @click="startSearch">Search</button>
    <div>
      <h1>{{ topic }} ( {{ page }} / {{ totalPage }} )</h1>

      <select v-model="perPage">
        <option :value="10">10</option>
        <option :value="15">15</option>
        <option :value="20">20</option>
      </select>

      <br>
      <button @click="gotoPage(page-1)">Prev</button>
      <button @click="gotoPage(page+1)">Next</button>

      <h2 v-if="loading">Loading...</h2>
      <ul>
        <li v-for="x in list" :key="x"> {{ x }} </li>
      </ul>
    </div>

  </div>
</template>

<script>
//import HelloWorld from './components/HelloWorld'
import axios from 'axios'

const searchGithub = (q, page, perPage) =>
  axios.get('https://api.github.com/search/repositories', {
    params: {
      q,
      page,
      per_page: perPage
    }
})
.then((resp) => resp.data)
//17.29

export default {

  name: 'app',
  data () {
    return {
      q: '',
      list: [],
      total: 0,
      topic: '',
      loading: false,
      page: 1,
      perPage: 10
    }
  },
  computed: {
    totalPage () {
      return Math.ceil(this.total / this.perPage)
    }
  },
  methods : {
    search () {
      if (this.loading) return
      this.loading = true
      const q = this.q
      searchGithub(q, this.page, this.perPage)
      .then((data) => {
        console.log(data)
        this.topic = q
        this.list = data.items.map((x) => x.full_name)
        this.total = data.total_count
        this.loading = false
      })
    },
    startSearch (){
      this.page = 1
      this.search()
    },
    gotoPage (page) {
      if (page <= 0) return
      if (page > this.totalPage) return
      this.page = page
      this.search()
    }

  },
  watch: {
    perPage () {
      this.search()
    }
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
