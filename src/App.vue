<template>
  <div id="app">
    <img src="./assets/logo.png">
    <br>
    <input ref="q">
    <button v-stream:click="q$">Search</button>
    <div>
      <h1>{{ topic }} ( {{ page }} / {{ totalPage }} )</h1>

      <select v-stream:change = "perPage$">
        <option :value="10">10</option>
        <option :value="15">15</option>
        <option :value="20">20</option>
      </select>


      <br>
      <button v-stream:click="prevClick$">Prev</button>
      <button v-stream:click="nextClick$">Next</button>

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

export default {
  name: 'app',
  domStreams: ['q$', 'prevClick$', 'nextClick$', 'perPage$'],
  subscriptions () {

    const page$ = new Subject()
    const loading$ = new BehaviorSubject(false)

    const q$$ = this.q$
      .map(() => this.$refs.q.value)
      .do (() => {console.log} )
      .do(() => { page$.next(1) })

    const page$$ = Observable.merge(
      page$,
      this.prevClick$.map(() => this.page - 1),
      this.nextClick$.map(() => this.page + 1),
    )

      .filter((page) => page > 0 && page <= this.totalPage)
      .startWith(1)

    const perPage$$ =  this.perPage$
      .map((e) => e.event.target.value)
      .startWith(10)

    const search$ = Observable.combineLatest(
      q$$,
      page$$,
      perPage$$
    )
        .debounceTime(1)
        //แปลง  q, page, perPage จาก array ให้เป็น obj
        .map(([q, page, perPage]) => ({  q, page, perPage }))
        .flatMap((x) => loading$.first(), (x, loading) => ({...x, loading}))
        .filter(({ loading }) => !loading)
        .do(() => { loading$.next(true) })
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
        .do(() => { loading$.next(false) })
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
        .startWith(0),
      page: search$
        .pluck('page')
        .startWith(1),
      loading: loading$
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
