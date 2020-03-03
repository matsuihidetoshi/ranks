<template>
  <div class="result">
    <h1>Results</h1>
    <div class="results-area">
      <div v-for="(result, id) in results" v-bind:key="id">
        <div class="single-result">
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
    <div id="chat-form">
      <button class="submit" v-on:click="createResult()">Post</button>
    </div>
  </div>
</template>
<script>
import { API, Auth, graphqlOperation} from "aws-amplify"
import { createResult } from "../graphql/mutations"
import { listResults } from "../graphql/queries"
import { getResult } from "../graphql/queries"
import { onCreateResult } from "../graphql/subscriptions"
import _ from 'lodash'

export default {
  name: 'Result',
  data () {
    return {
      scores: [99.9, 100.0, 150.1],
      successes: [1, 0, 1],
      result: null,
      results: [],
      owner: "",
      limit: 2 ** 31 - 1,
      user: null,
      test: null,
      name: "test name 2",
      groups: []
    }
  },
  mounted: function () {
    this.setOwner().then(
      this.displayResults()
    )
  },
  methods: {
    setOwner: async function () {
      this.user = await Auth.currentUserPoolUser(this.user)
      this.owner = this.user.username
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
    displayResults: async function () {
      let results = await API.graphql(graphqlOperation(
        listResults, {limit: this.limit}
      ))
      this.results = _.orderBy(results.data.listResults.items, 'id', 'desc').slice(0, 100)
      
      API.graphql(
        graphqlOperation(onCreateResult, {limit: this.limit, owner: this.owner})
      ).subscribe({
        next: (eventData) => {
          const result = eventData.value.data.onCreateResult
          const results = [...this.results, result]
          this.results = _.orderBy(results, 'id', 'desc').slice(0, 100)
        }
      })
      const user = await Auth.currentUserPoolUser(user)
      this.groups = user.signInUserSession.accessToken.payload['cognito:groups']
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
</style>