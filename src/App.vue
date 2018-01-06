<template>
  <div id="app">
    <img src="./assets/logo.png">
    <br>
    <input v-model="q">
    <button @click="q$.next(q)">Search</button>
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
      loading: false
    }
  },
  domStreams: ['prevClick$', 'nextClick$', 'perPage$'],
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

    this.q$ = new Subject()
    this.page$ = new Subject()

    const q$$ = this.q$
      .do(() => { this.page$.next(1) })

    const page$$ = Observable.merge(
      this.page$,
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
        .filter(() => !this.loading)
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
        .startWith(0),
      page: search$
        .pluck('page')
        .startWith(1)
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
