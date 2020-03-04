<template>
  <div class="result">
    <p>id: {{ id }}</p>
    <div v-for="(result, index) in results" v-bind:key="index">
      <p>id: {{ result.id }}</p>
    </div>
    <h1 v-if="(rank)">Rank</h1>
    <h1 v-else>Results</h1>
    <div class="results-area">
      <div v-for="(result, index) in results" v-bind:key="index">
        <div class="single-result" v-bind:class="{ selected: (id == result.id) }" v-on:click="selectId(result.id)">
          <div>id: {{ result.id }}</div>
          <div>name: {{ result.name }}</div>
          <div v-for="(score, index) in result.scores" v-bind:key="`score-${index}`">
            score{{ index + 1 }}: {{ score }}
          </div>
          <div v-for="(success, index) in result.successes" v-bind:key="`success_${index}`">
            success{{ index + 1 }}: {{ success }}
          </div>
        </div>
      </div>
    </div>
    <div>
      <button class="submit" v-on:click="createResult()">Post</button>
    </div>
    <div>
      <button class="submit" v-on:click="updateResult()">Update</button>
    </div>
    <div>
      <button class="submit" v-on:click="switchResult()">
        <span v-if="(rank)">Result</span>
        <span v-else>Rank</span>
      </button>
    </div>
  </div>
</template>
<script>
import { API, Auth, graphqlOperation } from "aws-amplify"//API:AppSync用 Auth:Cognito用 graphqlOperation:AppSyncのGraphQL用
import { createResult, updateResult } from "../graphql/mutations"//AppSync更新系
import { listResults, getResult } from "../graphql/queries"//AppSync取得系
import { onCreateResult, onUpdateResult } from "../graphql/subscriptions"//AppSyncリアルタイム取得(GraphQLのSubscriptions:WebSocket使ってる)系
import _ from 'lodash'

export default {
  name: 'Result',
  data () {
    return {
      id: null,//update用id
      scores: [19.8, 19.0, 10.1],//Resultのscores投稿用テストデータ
      successes: [1, 0, 1],//Resultのsuccesses投稿用テストデータ
      result: null,//Result投稿時格納ようの変数
      results: [],//Resultの配列
      owner: "",//Cognitoユーザー情報からusername格納
      limit: 2 ** 31 - 1,//appsyncのスキャンのリミット
      user: null,//Cognitoユーザー情報取得
      name: "test name 2",//Resultのname投稿用テストデータ
      groups: [],//グループ取得様配列（使ってない)
      rank: false//ランク表示か自分の結果表示かのフラグ
    }
  },
  mounted: function () {
    this.setOwner().then(//Cognitoログインユーザー情報取得できたら
      this.displayResults()//Resultsを取得してきて表示
    )
  },
  methods: {
    deduplicate: function (results) {//Resultsのupdateのsubscriptions用の、重複排除メソッド
      return results.filter(function(v1,i1,a1){ 
        return (a1.findIndex(function(v2){ 
          return (v1.id === v2.id) 
        }) === i1)
      })
    },
    selectId: function (id) {//一覧表示でクリックするとupdate用idを更新
      this.id = id
    },
    setOwner: async function () {//Cognitoユーザー情報取得
      this.user = await Auth.currentUserPoolUser(this.user)
      this.owner = this.user.username
    },
    switchResult: function () {//ランクか自分の一覧の切り替え
      if (this.rank == true) {
        this.rank = false
      } else {
        this.rank = true
      }
      this.displayResults()//切り替えたら、再表示
    },
    createResult: async function () {//新しくResult登録
      if (this.scores === [] || this.successes === []) return//フォーム入力想定して、値が空っぽだったら何もしない
      const result = {name: this.name, scores: this.scores, successes: this.successes}//GraphQLに投げるためのデータ整形
      try {
        this.name = ""//フォーム入力想定して、投稿するときにリセット
        this.scores = []//↑と同じ
        this.successes = []//↑と同じ
        await API.graphql(graphqlOperation(createResult, {input: result}))//Result作成API叩く
      } catch (error) {//エラー処理
        error
      }
    },
    updateResult: async function () {//既存のResult更新
      if (this.scores === [] || this.successes === []) return//フォーム入力想定して、値が空っぽだったら何もしない
      const result = {scores: [111.111, 222.222, 333.333], successes: [1, 1, 1], id: this.id}//更新なので、idも指定する
      try {
        this.scores = []//フォーム入力想定して、投稿するときにリセット
        this.scores = []//↑と同じ
        this.successes = []//↑と同じ
        await API.graphql(graphqlOperation(updateResult, {input: result}))//Result更新API叩く
      } catch (error) {
        error
      }
    },
    displayResults: async function () {//一覧表示メソッド
      let results = await API.graphql(graphqlOperation(//読み込み1発目の一覧取得
        listResults, {limit: this.limit}//limit与えているが、実質的に全件取得
      ))
      results = results.data.listResults.items//レスポンスから必要な部分抽出
      results.forEach(result => {//合計値計算
        result['sum'] = result['scores'].reduce((a,x) => a+=x,0)//'sum'というキーで結果に追加
      })
      this.results = _.orderBy(results, 'sum', 'desc').slice(0, 100)//合計値で並び替え
      if (!this.rank) {this.results = this.results.filter(result => result['owner'] == this.owner)}//ランクじゃなかったら、自分のResultだけ絞り込み
      
      API.graphql(
        graphqlOperation(onCreateResult, {limit: this.limit, owner: this.owner})//新規作成リアルタイム取得(多分ここでWebSocketのセッション張ってるイメージ)
      ).subscribe({
        next: (eventData) => {
          const result = eventData.value.data.onCreateResult//新しく追加されたResultを取得
          const results = [...this.results, result]//現状のResult一覧に取得した新規Resultを追加
          results.forEach(result => {//合計値計算
            result['sum'] = result['scores'].reduce((a,x) => a+=x,0)//'sum'というキーで結果に追加
          })
          this.results = _.orderBy(results, 'sum', 'desc').slice(0, 100)//合計値で並び替え
          if (!this.rank) {this.results = this.results.filter(result => result['owner'] == this.owner)}//ランクじゃなかったら、自分のResultだけ絞り込み
        }
      })

      API.graphql(
        graphqlOperation(onUpdateResult, {limit: this.limit, owner: this.owner})//更新リアルタイム取得(多分ここでWebSocketのセッション張ってるイメージ)
      ).subscribe({
        next: (eventData) => {
          const result = eventData.value.data.onUpdateResult//更新されたResultを取得
          const results = [...this.results, result]//現状のResult一覧に取得した更新Resultを追加
          results.forEach(result => {//合計値計算
            result['sum'] = result['scores'].reduce((a,x) => a+=x,0)//'sum'というキーで結果に追加
          })
          this.results = _.orderBy(results, 'sum', 'desc').slice(0, 100)//合計値で並び替え
          if (!this.rank) {this.results = this.results.filter(result => result['owner'] == this.owner)}//ランクじゃなかったら、自分のResultだけ絞り込み
          this.results = this.deduplicate(this.results)//単純に追加しちゃってるので重複排除
        }
      })
    },
    singleResult: async function (selectedNote) {//id指定で1件取得(使ってない)
      let result = await API.graphql(graphqlOperation(
        getResult, {id: selectedNote.id}
      ))
      this.result = result
    }
  }
}
</script>
<style scoped>
  textarea {
    height: 80px;
  }
  .form {
    font-size: 20px;
    border: solid 1px gray;
    border-radius: 5px;
    width: 300px;
  }
  .submit {
    width: 300px;
    font-size: 20px;
    border-radius: 5px;
    vertical-align: top;
  }
  .results-area {
    width: 300px;
    height: 300px;
    display: inline-block;
    overflow: scroll;
  }
  .single-result {
    padding: 10px 20px 10px 20px;
    background-color: #eeeeee;
    border-radius: 5px;
    margin: 10px 0 10px 0;
    overflow-wrap: break-word;
    white-space: pre-line;
  }
  .selected {
    background-color: #99ddee;
  }
</style>