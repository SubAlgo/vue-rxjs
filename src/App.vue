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
import { Observable, Subject, BehaviorSubject } from 'rxjs'
import axios from 'axios'

const searchGithub = (q, page, perPage) => Observable.fromPromise(
  axios.get('https://api.github.com/search/repositories', {
    params: {
      q,
      page,
      per_page: perPage
    }
}))
.map((resp) => resp.data)

/*
const searchGithub = (q, page, perPage) =>
  axios.get('https://api.github.com/search/repositories', {
    params: {
      q,
      page,
      per_page: perPage
    }
})
.then((resp) => resp.data)
*/


export default {

  name: 'app',
  data () {
    return {
      q: '',
      loading: false,
      page: 1,
      perPage: 10
    }
  },
  subscriptions () {
    /*
    Observable.defer(() => {
        //ถ้า loading อยู่ จะไปที่ never แทน ทำให้ api ไม่ถูก call ซ้ำ
        if (this.loading) return Observable.never()
        return Observable.of({ q: this.q, page: this.page, perPage: this.perPage })
      })
      51.03
      */

    /*
    Observable.of({a: 1, b: 2})
      .map((x) => x.a)
      .pluck('a')
      2บรรทัดบน .map กับ .pluck ได้ผลลัพธ์เหมือนกัน
    */

    const search$ = Observable.combineLatest(
      this.$watchAsObservable('q').pluck('newValue'),
      this.$watchAsObservable('page', { immediate: true }).pluck('newValue'),
      this.$watchAsObservable('perPage', { immediate: true }).pluck('newValue'),
    )
        //แปลง  q, page, perPage จาก array ให้เป็น obj
        .map(([q, page, perPage]) => ({  q, page, perPage }))
        .do(() => { this.loading = true })
        .flatMap(({ q, page, perPage }) =>
          searchGithub(q, page, perPage),
          ({ q, page, perPage }, data) => ({
             q,
             page,
             perPage,
             list: data.items,
             total: data.total_count
          }))
        //ย้าย this.loading = false มาใน finally เพื่อให้ทำคำสั่งนี้ทุกกรณีไม่ว่า success or error
        .do(() => { this.loading = false })
        .share()
    return {
      list: search$
        .pluck('list')
        .map((list) => list.map((x) => x.full_name)),
        //.map((resp) => resp.list.map((x) => x.full_name)),
      topic: search$
        .pluck('q'),
        //.map((resp) => resp.q),
      total: search$
        .pluck('total'),
        //.map((resp) => resp.total),
      totalPage: search$
        .map((resp) => Math.ceil(resp.total / resp.perPage))
        .startWith(0)
    }
  },
  /*
  computed: {
    totalPage () {
      return Math.ceil(this.total / this.perPage)
    }
  },
  */
  methods : {
    search () {

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
