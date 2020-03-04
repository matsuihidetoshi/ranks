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
import { API, Auth, graphqlOperation} from "aws-amplify"
import { createResult, updateResult } from "../graphql/mutations"
import { listResults } from "../graphql/queries"
import { getResult } from "../graphql/queries"
import { onCreateResult, onUpdateResult } from "../graphql/subscriptions"
import _ from 'lodash'

export default {
  name: 'Result',
  data () {
    return {
      id: null,
      scores: [19.8, 19.0, 10.1],
      successes: [1, 0, 1],
      result: null,
      results: [],
      owner: "",
      limit: 2 ** 31 - 1,
      user: null,
      name: "test name 2",
      groups: [],
      rank: false
    }
  },
  mounted: function () {
    this.setOwner().then(
      this.displayResults()
    )
  },
  methods: {
    deduplicate: function (results) {
      return results.filter(function(v1,i1,a1){ 
        return (a1.findIndex(function(v2){ 
          return (v1.id === v2.id) 
        }) === i1)
      })
    },
    selectId: function (id) {
      this.id = id
    },
    setOwner: async function () {
      this.user = await Auth.currentUserPoolUser(this.user)
      this.owner = this.user.username
    },
    switchResult: function () {
      if (this.rank == true) {
        this.rank = false
      } else {
        this.rank = true
      }
      this.displayResults()
    },
    createResult: async function () {
      if (this.scores === [] || this.successes === []) return
      const result = {name: this.name, scores: this.scores, successes: this.successes}
      try {
        this.name = ""
        this.scores = []
        this.successes = []
        await API.graphql(graphqlOperation(createResult, {input: result}))
      } catch (error) {
        error
      }
    },
    updateResult: async function () {
      if (this.scores === [] || this.successes === []) return
      const result = {scores: [111.111, 222.222, 333.333], successes: [1, 1, 1], id: this.id}
      try {
        this.scores = []
        this.successes = []
        await API.graphql(graphqlOperation(updateResult, {input: result}))
      } catch (error) {
        error
      }
    },
    displayResults: async function () {
      let results = await API.graphql(graphqlOperation(
        listResults, {limit: this.limit}
      ))
      results = results.data.listResults.items
      results.forEach(result => {
        result['sum'] = result['scores'].reduce((a,x) => a+=x,0)
      })
      this.results = _.orderBy(results, 'sum', 'desc').slice(0, 100)
      if (!this.rank) {this.results = this.results.filter(result => result['owner'] == this.owner)}
      
      API.graphql(
        graphqlOperation(onCreateResult, {limit: this.limit, owner: this.owner})
      ).subscribe({
        next: (eventData) => {
          const result = eventData.value.data.onCreateResult
          const results = [...this.results, result]
          results.forEach(result => {
            result['sum'] = result['scores'].reduce((a,x) => a+=x,0)
          })
          this.results = _.orderBy(results, 'sum', 'desc').slice(0, 100)
          if (!this.rank) {this.results = this.results.filter(result => result['owner'] == this.owner)}
        }
      })

      API.graphql(
        graphqlOperation(onUpdateResult, {limit: this.limit, owner: this.owner})
      ).subscribe({
        next: (eventData) => {
          const result = eventData.value.data.onUpdateResult
          const results = [...this.results, result]
          results.forEach(result => {
            result['sum'] = result['scores'].reduce((a,x) => a+=x,0)
          })
          this.results = _.orderBy(results, 'sum', 'desc').slice(0, 100)
          if (!this.rank) {this.results = this.results.filter(result => result['owner'] == this.owner)}
          this.results = this.deduplicate(this.results)
        }
      })
    },
    singleResult: async function (selectedNote) {
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